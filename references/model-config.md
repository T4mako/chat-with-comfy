# 模型注册表

`api/models.json` 是可扩展的模型注册表。`scripts/comfy_media.py` 从中读取模型名、工作流文件和可写节点映射；因此添加与现有五种交互方式兼容的 ComfyUI 模型时，通常不需要修改 Python。

运行以下命令查看当前注册的模型，并在每次修改配置后验证节点映射：

```powershell
python scripts/comfy_media.py models
python scripts/comfy_media.py validate
```

请求可用可选的 `model` 字段选择模型：

```json
{
  "mode": "t2i",
  "model": "my-z-image",
  "prompt": "A quiet snow-covered observatory at blue hour",
  "width": 1344,
  "height": 768
}
```

未设置 `model` 时，脚本使用该 `mode` 标记为 `"default": true` 的模型。每个 `mode` 必须且只能有一个默认模型。

## 新增同类模型

如果新模型与一个已注册工作流的输入结构相同，最简方式是在 `models` 下添加一个继承项，并通过 `overrides` 替换 ComfyUI 节点的输入。例如下面的模型沿用 `z-image-turbo` 的文本、尺寸、seed 和保存节点，只替换 UNet：

```json
"my-z-image": {
  "extends": "z-image-turbo",
  "description": "My locally installed Z-Image variant",
  "default_filename_prefix": "my-z-image",
  "overrides": [
    {
      "node": "57:28",
      "input": "unet_name",
      "value": "my_z_image_model.safetensors"
    }
  ]
}
```

`extends` 会继承父项的 `mode`、`workflow`、`bindings` 和其他设置；子项的 `overrides` 会追加到父项。派生模型默认不是默认模型，只有显式写 `"default": true` 才会成为该模式的默认项（并应同时取消原默认项）。

## 新增工作流

对于节点编号或输入结构不同的模型，将 ComfyUI 导出的 **API format** 工作流 JSON 放到 `api/`，然后创建完整 profile。下例展示 t2i 所需字段；其中节点 ID 与输入名须替换为新工作流的真实值：

```json
"my-checkpoint": {
  "description": "A custom text-to-image workflow",
  "mode": "t2i",
  "workflow": "my_checkpoint_t2i.json",
  "default_filename_prefix": "my-checkpoint",
  "bindings": {
    "prompt": { "node": "6", "input": "text" },
    "width": { "node": "5", "input": "width" },
    "height": { "node": "5", "input": "height" },
    "batch_size": { "node": "5", "input": "batch_size" },
    "seed": { "node": "3", "input": "seed" },
    "filename_prefix": { "node": "9", "input": "filename_prefix" }
  }
}
```

工作流路径必须是 `api/` 内的相对 `.json` 文件。`validate` 会在提交任务前检测配置引用的节点或输入是否已经失效。

## 各模式所需配置

所有 profile 都需要 `mode`、`workflow`、`default_filename_prefix` 和 `bindings`。每条 binding 的格式为 `{ "node": "节点 ID", "input": "输入名称" }`。

| `mode` | 必需的 `bindings` | 其他必需配置 |
| --- | --- | --- |
| `t2i` | `prompt`、`width`、`height`、`batch_size`、`seed`、`filename_prefix` | 无 |
| `i2i` | `prompt`、`seed`、`batch_size`、`filename_prefix` | `image_slots`，包含 1–10 个图像输入；可为可选图加 `enabled` binding |
| `t2v` | `prompt`、`duration`、`seed`、`aspect_ratio`、`megapixels`、`filename_prefix` | 无 |
| `i2v` | `prompt`、`duration`、`seed`、`aspect_ratio`、`megapixels`、`filename_prefix` | `image_slots`，且必须正好一个首帧输入 |
| `r2v` | `prompt`、`duration`、`seed`、`aspect_ratio`、`megapixels`、`filename_prefix` | `reference_images`，见下一节 |

`i2i` 的一个 slot 形如 `{ "image": { "node": "42", "input": "image" } }`。若该图是可选的，再加上 `"enabled": { "node": "148", "input": "value" }`；脚本会按照上传图像的顺序设置图像和开关。

`r2v` 的 `reference_images` 需要声明参考图生成器节点、可克隆的图像加载节点和字段前缀。内置 MiniMax 示例可直接作为模板：

```json
"reference_images": {
  "generator_node": "136",
  "input_prefix": "ref_images.ref_image_",
  "template_node": "137",
  "load_input": "image",
  "initial_load_nodes": ["137", "139"],
  "max_images": 9
}
```

`prompt_format` 可设为 `plain`（默认）、`h3_base`（仅 `t2v`/`i2v`）或 `h3_reference`（仅 `r2v`）。选择 `plain` 时，prompt 原样传入工作流；H3 格式会验证相应的完整提示词字段。
