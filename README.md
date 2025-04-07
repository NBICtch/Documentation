# Loverse AI Documentation

## Contents
- [Introduction](#introduction)
- [Technical Overview](#technical-overview)
- [Fooocus (Old Workflow)](#fooocus)
- [ComfyUI (New Workflow)](#comfyui)
- [Hand Fixing API](#hand-fixing-api)
- [ComfyUI & Hand Fixing Sample](#comfyui-hand-fixing)
- [Topaz Upscaling API](#topaz-upscaling-api)


## Introduction

Loverse AI is a creative platform that specializes in transforming ordinary photos into magical, personalized portraits. Our mission is to empower children and families by turning their photos into unique artworks that spark imagination and build confidence.

### What We Do
- Create beautiful, personalized portraits in seconds
- Offer a wide range of themes including:
  - Princesses and fantasy characters for girls
  - Superheroes and fantasy characters for boys
  - Royal portraits for mothers
  - Custom pet portraits
- Provide both digital and physical canvas options

## Technical Overview

### Technology Stack
- **Frontend**: Shopify
- **Backend**: AI Serverless API architecture with RunPod
- **Storage**: AWS S3 for image storage
- **AI Models**: Fooocus with SDXL (Old) and ComfyUI with Flux (New)

## Fooocus

This is the current workflow used by Loverse. It utilizes [Fooocus](https://github.com/lllyasviel/Fooocus) as the main engine and is served through the [Runpod serverless API](https://www.runpod.io/console/serverless/user/endpoint/po7ducp1sgd6xs).

### API Documentation

#### Authentication
The API requires a RunPod API token for authentication. Include it in the Authorization header:
```
Authorization: Bearer <your_runpod_token>
```

#### Image Generation Endpoint

**Endpoint**: `POST /run`

**Headers**:
```
Content-Type: application/json
Authorization: Bearer <your_runpod_token>
```

**Request Body**:
```json
{
    "input": {
        "api_name": "img2img2",
        "prompt": "your positive prompt",
        "negative_prompt": "your negative prompt",
        "style_selections": ["your style selections"],
        "performance_selection": "Speed",
        "aspect_ratios_selection": "896*1152",
        "image_number": 1,
        "image_seed": -1,
        "sharpness": 4,
        "guidance_scale": 6,
        "base_model_name": "OpenDalleV1.1.safetensors",
        "image_prompts": [
            {
                "cn_img": "<base64_encoded_image>",
                "cn_stop": 0.5,
                "cn_weight": 0.5,
                "cn_type": "ImagePrompt"
            },
            {
                "cn_img": "<base64_encoded_image>",
                "cn_stop": 0.5,
                "cn_weight": 0.5,
                "cn_type": "PyraCanny"
            },
            {
                "cn_img": "<base64_encoded_image>",
                "cn_stop": 1.0,
                "cn_weight": 0.6,
                "cn_type": "FaceSwap"
            }
        ]
    }
}
```

**Response**:
```json
{
    "id": "job_id",
    "status": "IN_QUEUE"
}
```

#### Status Check Endpoint

**Endpoint**: `GET /status/{job_id}`

**Headers**:
```
Authorization: Bearer <your_runpod_token>
```

**Response**:
```json
{
    "status": "COMPLETED",
    "output": [
        {
            "base64": "<base64_encoded_image>"
        }
    ]
}
```

#### Input Parameters

1. **Required Parameters**:
    - `api_name`: Must be "img2img2"
    - `prompt`: The positive prompt describing the desired output
    - `negative_prompt`: The negative prompt describing what you don't want in the output image
    - `style_selections`: Array of style names to apply to the generation. These styles are predefined combinations of prompts and settings that can be used to achieve specific artistic looks or effects. Multiple styles can be combined.
    - `image_prompts`: Array of control images with their respective types and weights
    - `ImagePrompt` and `PyraCanny`: Use product image as input
    - `FaceSwap`: Use user selfie as input

#### Input Data Source
The input data for the API can be obtained from the shopify product metafield configuration. Here's a reference image showing the available input fields:

![Fooocus Metafield Configuration](src/fooocus_metafield.png)

This image shows all the available configuration options that can be used in the API request. The fields in the image correspond to the parameters you can use in the API request.

#### Output

The API returns a job ID that can be used to check the status of the generation. Once completed, the response will include base64-encoded images in the output array.


## ComfyUI

We are currently migrating from Fooocus to [ComfyUI](https://github.com/comfyanonymous/ComfyUI). Unlike our previous workflow, this new workflow uses Flux instead of SDXL as the base AI model. The ComfyUI is already deployed as a serverless API on RunPod and can be accessed at [RunPod Endpoint](https://www.runpod.io/console/serverless/user/endpoint/e5agk6o3v0hcl1).

### Hair Depth Presets
The depth image used in the generation process varies based on the user's hair type. Different hair types require different depth maps to achieve optimal results:

![Hair Depth Presets](src/Hair%20Depth%20Preset.png)

The depth image selection follows these 9 hair type categories:
1. Straight Short Hair
2. Straight Medium Hair
3. Straight Long Hair
4. Curly Short Hair
5. Curly Medium Hair
6. Curly Long Hair
7. Afro Short Hair
8. Afro Medium Hair
9. Afro Long Hair

Each hair type has its own optimized depth map to ensure the best generation results. The system will automatically select the appropriate depth image based on the hair type and length information provided in the user's form submission on the website.

### Shopify Products Metafields Configuration
The input data for the API can be obtained from the Shopify product metafield configuration. Here's a reference image showing the available fields:

![ComfyUI Metafield Configuration](src/comfyui_metafield.png)

This image shows all the available configuration options that can be used in the API request. The fields in the image correspond to the parameters you can use in the API request, including:
- Prompt templates
- Depth image selection
- Other generation parameters

### API Documentation

#### Authentication
The API requires a RunPod API token for authentication. Include it in the Authorization header:
```
Authorization: Bearer <your_runpod_token>
```

#### Image Generation Endpoint

**Endpoint**: `POST /run`

**Headers**:
```
Content-Type: application/json
Authorization: Bearer <your_runpod_token>
```

**Request Body**:
```json
{
    "input": {
        "workflow": {
            // Workflow configuration loaded from ComfyUI-Pulid-ACE.json
            // Only modify these specific nodes:
            "107": {
                "inputs": {
                    "text": "your prompt"  // Your generation prompt
                }
            },
            "111": {
                "inputs": {
                    "noise_seed": 123456789  // Random seed or specific seed
                }
            },
            "113": {
                "inputs": {
                    "batch_size": 4  // Number of images to generate
                }
            }
            // Do not modify other workflow nodes unless you understand the full structure
        },
        "images": [
            {
                "name": "user.png",
                "image": "<base64_encoded_image_or_image_url>"  // Can be base64 string or direct URL to image
            },
            {
                "name": "depth_image.png",
                "image": "<base64_encoded_image_or_image_url>"  // Select from 9 hair type depth presets
            }
        ]
    }
}
```

The workflow configuration is loaded from the [`ComfyUI-Pulid-ACE.json`](src/ComfyUI-Pulid-ACE.json) file, which contains the complete node setup for the image generation pipeline. When making a request, you should:
1. Load the JSON file
2. Modify only the specified nodes (107, 111, 113)
3. Keep all other nodes and settings unchanged to ensure proper functionality
4. Select the appropriate depth image based on the user's hair type (see Hair Depth Presets section above)

Example loading and modifying the workflow in Python:
```python
# Load the workflow
with open('ComfyUI-Pulid-ACE.json', 'r') as f:
    workflow = json.load(f)

# Modify only these parameters
workflow["107"]["inputs"]["text"] = "your prompt"
workflow["111"]["inputs"]["noise_seed"] = "random seed"
workflow["113"]["inputs"]["batch_size"] = 4
```

**Response**:
```json
{
    "id": "job_id",
    "status": "IN_QUEUE"
}
```

#### Status Check Endpoint

**Endpoint**: `GET /status/{job_id}`

**Headers**:
```
Authorization: Bearer <your_runpod_token>
```

**Response**:
```json
{
    "status": "COMPLETED",
    "output": {
        "message": [
            "<s3_url_to_generated_image_1>",
            "<s3_url_to_generated_image_2>",
            "<s3_url_to_generated_image_3>",
            "<s3_url_to_generated_image_4>"
        ]
    }
}
```

#### Input Parameters

1. **Required Parameters**:
    - `workflow`: JSON configuration defining the generation process
    - `images`: Array of two input images:
      - User image (`user.png`)
      - Depth image (`depth_image.png`)

2. **Workflow Parameters**:
    - `prompt`: Text describing the desired output (node 107)
    - `seed`: Random seed for generation (node 111)
    - `batch_size`: Number of images to generate (node 113)

#### Output

The API returns:
1. A job ID that can be used to check the status
2. Once completed, S3 URLs to the generated images (number of URLs matches batch_size)

## Hand Fixing API

The Hand Fixing API is a specialized tool designed to improve AI-generated hands and fingers in images. This API is served through the [Runpod serverless API](https://www.runpod.io/console/serverless/user/endpoint/rnhxxj0c37e0le) and is particularly useful when the main generation models (Fooocus or ComfyUI) produce suboptimal hand renderings. Users can trigger hand regeneration through a button on the website after reviewing their initial generation results.

### API Documentation

#### Authentication
The API requires a RunPod API token for authentication. Include it in the Authorization header:
```
Authorization: Bearer <your_runpod_token>
```

#### Hand Fixing Endpoint

**Endpoint**: `POST /run`

**Headers**:
```
Content-Type: application/json
Authorization: Bearer <your_runpod_token>
```

**Request Body**:
```json
{
    "input": {
        "image": "<base64_encoded_image>",
        "mask": "<base64_encoded_mask>",
        "prompt": "young hand",
        "num_outputs": 1
    }
}
```

**Parameters**:
- `image`: Base64-encoded image containing the problematic hand(s)
- `mask`: Base64-encoded mask image highlighting the hand area to be fixed
- `prompt`: Description of the desired hand appearance (default: "young hand")
- `num_outputs`: Number of variations to generate (default: 1)

#### Request Examples

**Submit Job (curl)**:
```bash
curl -X POST "${RUNPOD_BASE_URL}/run" \
  -H "Authorization: Bearer ${RUNPOD_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "input": {
      "image": "<base64_encoded_image>",
      "mask": "<base64_encoded_mask>",
      "prompt": "young feminine hand",
      "num_outputs": 1
    }
  }'
```

**Response**:
```json
{
    "id": "xxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxx",
    "status": "IN_QUEUE"
}
```

**Check Status (curl)**:
```bash
curl -X GET "${RUNPOD_BASE_URL}/status/xxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxx" \
  -H "Authorization: Bearer ${RUNPOD_TOKEN}"
```

**Response (In Progress)**:
```json
{
    "id": "xxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxx",
    "status": "IN_PROGRESS"
}
```

**Response (Completed)**:
```json
{
    "id": "xxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxx",
    "status": "COMPLETED",
    "output": {
        "images": [
            "<base64_encoded_image>"
        ]
    }
}
```

**Response (Failed)**:
```json
{
    "id": "xxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxx",
    "status": "FAILED",
    "error": "Error message details"
}
```

The Hand Fixing API is designed to be integrated into the main workflow where users can:
1. Review their generated image
2. If they notice issues with hands/fingers
3. Click a regenerate button that will:
   - Retrieve hand mask for specific product from [comfyui metafield](src\comfyui_metafield.png)
   - Call this API to fix the hands
   - Replace the original image with the fixed version

This feature ensures that all generated images maintain high quality, particularly in challenging areas like hand rendering.

## ComfyUI Hand Fixing

We have a sample UI for making generation requests to ComfyUI and the hand fixing endpoint. You can try it out [here](https://huggingface.co/spaces/LLVVSE/Simple-ComfyUI).

## Topaz Upscaling API

We use [Topaz Labs' Enhance API](https://www.topazlabs.com/enhance-api) for high-quality image upscaling. This API provides industry-leading AI models for image enhancement and upscaling, ensuring our generated portraits maintain their quality even at larger sizes.

For detailed API documentation and implementation details, please refer to the [official Topaz Labs API documentation](https://developer.topazlabs.com/docs/welcome).

### Sample UI

We provide a simple interface for testing the Topaz upscaling API at our [Hugging Face Space](https://huggingface.co/spaces/LLVVSE/Topaz-Upscale). This UI allows you to:
1. Upload an image
2. Select desired output dimensions
3. Preview the upscaled result
4. Download the enhanced image
