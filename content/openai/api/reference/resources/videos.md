> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

> The Sora 2 models and Videos API were shut down on September 24, 2026 and are no longer available. No one-to-one replacement API is available. This page is retained for historical reference. See the [deprecation notice](https://developers.openai.com/api/docs/deprecations#2026-03-24-sora-2-video-generation-models-and-videos-api).

# Videos

## Create a video

**post** `/videos`

Create a new video generation job from a prompt and optional reference assets.

### Body Parameters

- `prompt: string`

  Text prompt that describes the video to generate.

  `minLength: 1`, `maxLength: 32000`

- `input_reference: optional ImageInputReferenceParam`

  Optional reference object that guides generation. Provide exactly one of `image_url` or `file_id`.

  - `file_id: optional string`

  - `image_url: optional string`

    A fully qualified URL or base64-encoded data URL.

    `maxLength: 20971520`, `format: uri`

- `model: optional VideoModel`

  The video generation model to use (allowed values: sora-2, sora-2-pro). Defaults to `sora-2`.

  - `string`

  - `"sora-2" or "sora-2-pro" or "sora-2-2025-10-06" or 2 more`

    - `"sora-2"`

    - `"sora-2-pro"`

    - `"sora-2-2025-10-06"`

    - `"sora-2-pro-2025-10-06"`

    - `"sora-2-2025-12-08"`

- `seconds: optional VideoSeconds`

  Clip duration in seconds (allowed values: 4, 8, 12). Defaults to 4 seconds.

  - `"4"`

  - `"8"`

  - `"12"`

- `size: optional VideoSize`

  Output resolution formatted as width x height (allowed values: 720x1280, 1280x720, 1024x1792, 1792x1024). Defaults to 720x1280.

  - `"720x1280"`

  - `"1280x720"`

  - `"1024x1792"`

  - `"1792x1024"`

### Returns

- `Video object { id, completed_at, created_at, 10 more }`

  Structured information describing a generated video job.

  - `id: string`

    Unique identifier for the video job.

  - `completed_at: integer or null`

    Unix timestamp (seconds) for when the job completed, if finished.

    `format: unixtime`

  - `created_at: integer`

    Unix timestamp (seconds) for when the job was created.

    `format: unixtime`

  - `error: VideoCreateError or null`

    Error payload that explains why generation failed, if applicable.

    - `code: string`

      A machine-readable error code that was returned.

    - `message: string`

      A human-readable description of the error that was returned.

    - `headers: optional map[string]`

      The Retry-After and Retry-After-Ms headers returned with the original error, if any.

    - `misalignment: optional object { detailed_explanation, error_type, review_target, steer }`

      - `detailed_explanation: optional string`

        The public explanation for this block.

      - `error_type: optional string or "potentially_unintended_data_transfer" or "potentially_unintended_data_access" or "potentially_unintended_destructive_activity" or "other"`

        An optional classification; clients must accept additional values.

        - `string`

        - `SafetyAlertErrorType = "potentially_unintended_data_transfer" or "potentially_unintended_data_access" or "potentially_unintended_destructive_activity" or "other"`

          An optional classification; clients must accept additional values.

          - `"potentially_unintended_data_transfer"`

          - `"potentially_unintended_data_access"`

          - `"potentially_unintended_destructive_activity"`

          - `"other"`

      - `review_target: optional string or null`

        An opaque target for explicitly continuing this review, or null when unavailable.

        `pattern: ^[A-Za-z0-9._~:-]+$`, `minLength: 1`, `maxLength: 96`

      - `steer: optional object { message }`

        An optional public continuation instruction.

        - `message: string`

          The public continuation instruction.

    - `param: optional string or null`

      The request parameter associated with the error, or null if none was recorded.

    - `type: optional string or null`

      The original error type, or null if it was not recorded.

  - `expires_at: integer or null`

    Unix timestamp (seconds) for when the downloadable assets expire, if set.

    `format: unixtime`

  - `model: VideoModel`

    The video generation model that produced the job.

    - `string`

    - `"sora-2" or "sora-2-pro" or "sora-2-2025-10-06" or 2 more`

      - `"sora-2"`

      - `"sora-2-pro"`

      - `"sora-2-2025-10-06"`

      - `"sora-2-pro-2025-10-06"`

      - `"sora-2-2025-12-08"`

  - `object: "video" (default: "video")`

    The object type, which is always `video`.

    - `"video"`

  - `progress: integer`

    Approximate completion percentage for the generation task.

  - `prompt: string or null`

    The prompt that was used to generate the video.

  - `remixed_from_video_id: string or null`

    Identifier of the source video if this video is a remix.

  - `seconds: string`

    Duration of the generated clip in seconds. For extensions, this is the stitched total duration.

  - `size: VideoSize`

    The resolution of the generated video.

    - `"720x1280"`

    - `"1280x720"`

    - `"1024x1792"`

    - `"1792x1024"`

  - `status: "queued" or "in_progress" or "completed" or "failed"`

    Current lifecycle status of the video job.

    - `"queued"`

    - `"in_progress"`

    - `"completed"`

    - `"failed"`

### Example

```http
curl https://api.openai.com/v1/videos \
    -H 'Content-Type: application/json' \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -d '{
          "prompt": "x"
        }'
```

#### 200 — Example

```json
{
  "id": "id",
  "completed_at": 0,
  "created_at": 0,
  "error": {
    "code": "code",
    "message": "message",
    "headers": {
      "foo": "string"
    },
    "misalignment": {
      "detailed_explanation": "detailed_explanation",
      "error_type": "potentially_unintended_data_transfer",
      "review_target": "review_target",
      "steer": {
        "message": "message"
      }
    },
    "param": "param",
    "type": "type"
  },
  "expires_at": 0,
  "model": "sora-2",
  "object": "video",
  "progress": 0,
  "prompt": "prompt",
  "remixed_from_video_id": "remixed_from_video_id",
  "seconds": "string",
  "size": "720x1280",
  "status": "queued"
}
```

### Example

```http
curl https://api.openai.com/v1/videos \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -F "model=sora-2" \
  -F "prompt=A calico cat playing a piano on stage"
```

#### Response example

```json
{
  "id": "video_123",
  "object": "video",
  "model": "sora-2",
  "status": "queued",
  "progress": 0,
  "created_at": 1712697600,
  "size": "1024x1792",
  "seconds": "8",
  "prompt": "A calico cat playing a piano on stage",
  "completed_at": null,
  "expires_at": null,
  "error": null,
  "remixed_from_video_id": null
}
```

## Create a character

**post** `/videos/characters`

Create a character from an uploaded video.

### Returns

- `id: string or null`

  Identifier for the character creation cameo.

- `created_at: integer`

  Unix timestamp (in seconds) when the character was created.

  `format: unixtime`

- `name: string or null`

  Display name for the character.

### Example

```http
curl https://api.openai.com/v1/videos/characters \
    -H 'Content-Type: multipart/form-data' \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -F name=x \
    -F 'video=@/path/to/video'
```

#### 200 — Example

```json
{
  "id": "id",
  "created_at": 0,
  "name": "name"
}
```

## Delete a video

**delete** `/videos/{video_id}`

Permanently delete a completed or failed video and its stored assets.

### Path Parameters

- `video_id: string`

  The identifier of the video to delete.

### Returns

- `id: string`

  Identifier of the deleted video.

- `deleted: boolean`

  Indicates that the video resource was deleted.

- `object: "video.deleted" (default: "video.deleted")`

  The object type that signals the deletion response.

  - `"video.deleted"`

### Example

```http
curl https://api.openai.com/v1/videos/$VIDEO_ID \
    -X DELETE \
    -H "Authorization: Bearer $OPENAI_API_KEY"
```

#### 200 — Example

```json
{
  "id": "id",
  "deleted": true,
  "object": "video.deleted"
}
```

## Download video content

**get** `/videos/{video_id}/content`

Download the generated video bytes or a derived preview asset.

Streams the rendered video content for the specified video job.

### Path Parameters

- `video_id: string`

  The identifier of the video whose media to download.

### Query Parameters

- `variant: optional "video" or "thumbnail" or "spritesheet"`

  Which downloadable asset to return. Defaults to the MP4 video.

  - `"video"`

  - `"thumbnail"`

  - `"spritesheet"`

### Returns

`file`

### Example

```http
curl https://api.openai.com/v1/videos/$VIDEO_ID/content \
    -H "Authorization: Bearer $OPENAI_API_KEY"
```

## Create a video edit

**post** `/videos/edits`

Create a new video generation job by editing a source video or existing generated video.

### Body Parameters

- `prompt: string`

  Text prompt that describes how to edit the source video.

  `minLength: 1`, `maxLength: 32000`

- `video: object { id }`

  Reference to the completed video to edit.

  - `id: string`

    The identifier of the completed video.

### Returns

- `Video object { id, completed_at, created_at, 10 more }`

  Structured information describing a generated video job.

  - `id: string`

    Unique identifier for the video job.

  - `completed_at: integer or null`

    Unix timestamp (seconds) for when the job completed, if finished.

    `format: unixtime`

  - `created_at: integer`

    Unix timestamp (seconds) for when the job was created.

    `format: unixtime`

  - `error: VideoCreateError or null`

    Error payload that explains why generation failed, if applicable.

    - `code: string`

      A machine-readable error code that was returned.

    - `message: string`

      A human-readable description of the error that was returned.

    - `headers: optional map[string]`

      The Retry-After and Retry-After-Ms headers returned with the original error, if any.

    - `misalignment: optional object { detailed_explanation, error_type, review_target, steer }`

      - `detailed_explanation: optional string`

        The public explanation for this block.

      - `error_type: optional string or "potentially_unintended_data_transfer" or "potentially_unintended_data_access" or "potentially_unintended_destructive_activity" or "other"`

        An optional classification; clients must accept additional values.

        - `string`

        - `SafetyAlertErrorType = "potentially_unintended_data_transfer" or "potentially_unintended_data_access" or "potentially_unintended_destructive_activity" or "other"`

          An optional classification; clients must accept additional values.

          - `"potentially_unintended_data_transfer"`

          - `"potentially_unintended_data_access"`

          - `"potentially_unintended_destructive_activity"`

          - `"other"`

      - `review_target: optional string or null`

        An opaque target for explicitly continuing this review, or null when unavailable.

        `pattern: ^[A-Za-z0-9._~:-]+$`, `minLength: 1`, `maxLength: 96`

      - `steer: optional object { message }`

        An optional public continuation instruction.

        - `message: string`

          The public continuation instruction.

    - `param: optional string or null`

      The request parameter associated with the error, or null if none was recorded.

    - `type: optional string or null`

      The original error type, or null if it was not recorded.

  - `expires_at: integer or null`

    Unix timestamp (seconds) for when the downloadable assets expire, if set.

    `format: unixtime`

  - `model: VideoModel`

    The video generation model that produced the job.

    - `string`

    - `"sora-2" or "sora-2-pro" or "sora-2-2025-10-06" or 2 more`

      - `"sora-2"`

      - `"sora-2-pro"`

      - `"sora-2-2025-10-06"`

      - `"sora-2-pro-2025-10-06"`

      - `"sora-2-2025-12-08"`

  - `object: "video" (default: "video")`

    The object type, which is always `video`.

    - `"video"`

  - `progress: integer`

    Approximate completion percentage for the generation task.

  - `prompt: string or null`

    The prompt that was used to generate the video.

  - `remixed_from_video_id: string or null`

    Identifier of the source video if this video is a remix.

  - `seconds: string`

    Duration of the generated clip in seconds. For extensions, this is the stitched total duration.

  - `size: VideoSize`

    The resolution of the generated video.

    - `"720x1280"`

    - `"1280x720"`

    - `"1024x1792"`

    - `"1792x1024"`

  - `status: "queued" or "in_progress" or "completed" or "failed"`

    Current lifecycle status of the video job.

    - `"queued"`

    - `"in_progress"`

    - `"completed"`

    - `"failed"`

### Example

```http
curl https://api.openai.com/v1/videos/edits \
    -H 'Content-Type: application/json' \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -d '{
          "prompt": "x",
          "video": {
            "id": "video_123"
          }
        }'
```

#### 200 — Example

```json
{
  "id": "id",
  "completed_at": 0,
  "created_at": 0,
  "error": {
    "code": "code",
    "message": "message",
    "headers": {
      "foo": "string"
    },
    "misalignment": {
      "detailed_explanation": "detailed_explanation",
      "error_type": "potentially_unintended_data_transfer",
      "review_target": "review_target",
      "steer": {
        "message": "message"
      }
    },
    "param": "param",
    "type": "type"
  },
  "expires_at": 0,
  "model": "sora-2",
  "object": "video",
  "progress": 0,
  "prompt": "prompt",
  "remixed_from_video_id": "remixed_from_video_id",
  "seconds": "string",
  "size": "720x1280",
  "status": "queued"
}
```

## Create a video extension

**post** `/videos/extensions`

Create an extension of a completed video.

### Body Parameters

- `prompt: string`

  Updated text prompt that directs the extension generation.

  `minLength: 1`, `maxLength: 32000`

- `seconds: VideoSeconds`

  Length of the newly generated extension segment in seconds (allowed values: 4, 8, 12, 16, 20).

  - `"4"`

  - `"8"`

  - `"12"`

- `video: object { id }`

  Reference to the completed video to extend.

  - `id: string`

    The identifier of the completed video.

### Returns

- `Video object { id, completed_at, created_at, 10 more }`

  Structured information describing a generated video job.

  - `id: string`

    Unique identifier for the video job.

  - `completed_at: integer or null`

    Unix timestamp (seconds) for when the job completed, if finished.

    `format: unixtime`

  - `created_at: integer`

    Unix timestamp (seconds) for when the job was created.

    `format: unixtime`

  - `error: VideoCreateError or null`

    Error payload that explains why generation failed, if applicable.

    - `code: string`

      A machine-readable error code that was returned.

    - `message: string`

      A human-readable description of the error that was returned.

    - `headers: optional map[string]`

      The Retry-After and Retry-After-Ms headers returned with the original error, if any.

    - `misalignment: optional object { detailed_explanation, error_type, review_target, steer }`

      - `detailed_explanation: optional string`

        The public explanation for this block.

      - `error_type: optional string or "potentially_unintended_data_transfer" or "potentially_unintended_data_access" or "potentially_unintended_destructive_activity" or "other"`

        An optional classification; clients must accept additional values.

        - `string`

        - `SafetyAlertErrorType = "potentially_unintended_data_transfer" or "potentially_unintended_data_access" or "potentially_unintended_destructive_activity" or "other"`

          An optional classification; clients must accept additional values.

          - `"potentially_unintended_data_transfer"`

          - `"potentially_unintended_data_access"`

          - `"potentially_unintended_destructive_activity"`

          - `"other"`

      - `review_target: optional string or null`

        An opaque target for explicitly continuing this review, or null when unavailable.

        `pattern: ^[A-Za-z0-9._~:-]+$`, `minLength: 1`, `maxLength: 96`

      - `steer: optional object { message }`

        An optional public continuation instruction.

        - `message: string`

          The public continuation instruction.

    - `param: optional string or null`

      The request parameter associated with the error, or null if none was recorded.

    - `type: optional string or null`

      The original error type, or null if it was not recorded.

  - `expires_at: integer or null`

    Unix timestamp (seconds) for when the downloadable assets expire, if set.

    `format: unixtime`

  - `model: VideoModel`

    The video generation model that produced the job.

    - `string`

    - `"sora-2" or "sora-2-pro" or "sora-2-2025-10-06" or 2 more`

      - `"sora-2"`

      - `"sora-2-pro"`

      - `"sora-2-2025-10-06"`

      - `"sora-2-pro-2025-10-06"`

      - `"sora-2-2025-12-08"`

  - `object: "video" (default: "video")`

    The object type, which is always `video`.

    - `"video"`

  - `progress: integer`

    Approximate completion percentage for the generation task.

  - `prompt: string or null`

    The prompt that was used to generate the video.

  - `remixed_from_video_id: string or null`

    Identifier of the source video if this video is a remix.

  - `seconds: string`

    Duration of the generated clip in seconds. For extensions, this is the stitched total duration.

  - `size: VideoSize`

    The resolution of the generated video.

    - `"720x1280"`

    - `"1280x720"`

    - `"1024x1792"`

    - `"1792x1024"`

  - `status: "queued" or "in_progress" or "completed" or "failed"`

    Current lifecycle status of the video job.

    - `"queued"`

    - `"in_progress"`

    - `"completed"`

    - `"failed"`

### Example

```http
curl https://api.openai.com/v1/videos/extensions \
    -H 'Content-Type: application/json' \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -d '{
          "prompt": "x",
          "seconds": "4",
          "video": {
            "id": "video_123"
          }
        }'
```

#### 200 — Example

```json
{
  "id": "id",
  "completed_at": 0,
  "created_at": 0,
  "error": {
    "code": "code",
    "message": "message",
    "headers": {
      "foo": "string"
    },
    "misalignment": {
      "detailed_explanation": "detailed_explanation",
      "error_type": "potentially_unintended_data_transfer",
      "review_target": "review_target",
      "steer": {
        "message": "message"
      }
    },
    "param": "param",
    "type": "type"
  },
  "expires_at": 0,
  "model": "sora-2",
  "object": "video",
  "progress": 0,
  "prompt": "prompt",
  "remixed_from_video_id": "remixed_from_video_id",
  "seconds": "string",
  "size": "720x1280",
  "status": "queued"
}
```

## Retrieve a character

**get** `/videos/characters/{character_id}`

Fetch a character.

### Path Parameters

- `character_id: string`

  The identifier of the character to retrieve.

### Returns

- `id: string or null`

  Identifier for the character creation cameo.

- `created_at: integer`

  Unix timestamp (in seconds) when the character was created.

  `format: unixtime`

- `name: string or null`

  Display name for the character.

### Example

```http
curl https://api.openai.com/v1/videos/characters/$CHARACTER_ID \
    -H "Authorization: Bearer $OPENAI_API_KEY"
```

#### 200 — Example

```json
{
  "id": "id",
  "created_at": 0,
  "name": "name"
}
```

## List videos

**get** `/videos`

List recently generated videos for the current project.

### Query Parameters

- `after: optional string`

  Identifier for the last item from the previous pagination request

- `limit: optional integer`

  Number of items to retrieve

  `minimum: 0`, `maximum: 100`

- `order: optional "asc" or "desc"`

  Sort order of results by timestamp. Use `asc` for ascending order or `desc` for descending order.

  - `"asc"`

  - `"desc"`

### Returns

- `data: array of Video`

  A list of items

  - `id: string`

    Unique identifier for the video job.

  - `completed_at: integer or null`

    Unix timestamp (seconds) for when the job completed, if finished.

    `format: unixtime`

  - `created_at: integer`

    Unix timestamp (seconds) for when the job was created.

    `format: unixtime`

  - `error: VideoCreateError or null`

    Error payload that explains why generation failed, if applicable.

    - `code: string`

      A machine-readable error code that was returned.

    - `message: string`

      A human-readable description of the error that was returned.

    - `headers: optional map[string]`

      The Retry-After and Retry-After-Ms headers returned with the original error, if any.

    - `misalignment: optional object { detailed_explanation, error_type, review_target, steer }`

      - `detailed_explanation: optional string`

        The public explanation for this block.

      - `error_type: optional string or "potentially_unintended_data_transfer" or "potentially_unintended_data_access" or "potentially_unintended_destructive_activity" or "other"`

        An optional classification; clients must accept additional values.

        - `string`

        - `SafetyAlertErrorType = "potentially_unintended_data_transfer" or "potentially_unintended_data_access" or "potentially_unintended_destructive_activity" or "other"`

          An optional classification; clients must accept additional values.

          - `"potentially_unintended_data_transfer"`

          - `"potentially_unintended_data_access"`

          - `"potentially_unintended_destructive_activity"`

          - `"other"`

      - `review_target: optional string or null`

        An opaque target for explicitly continuing this review, or null when unavailable.

        `pattern: ^[A-Za-z0-9._~:-]+$`, `minLength: 1`, `maxLength: 96`

      - `steer: optional object { message }`

        An optional public continuation instruction.

        - `message: string`

          The public continuation instruction.

    - `param: optional string or null`

      The request parameter associated with the error, or null if none was recorded.

    - `type: optional string or null`

      The original error type, or null if it was not recorded.

  - `expires_at: integer or null`

    Unix timestamp (seconds) for when the downloadable assets expire, if set.

    `format: unixtime`

  - `model: VideoModel`

    The video generation model that produced the job.

    - `string`

    - `"sora-2" or "sora-2-pro" or "sora-2-2025-10-06" or 2 more`

      - `"sora-2"`

      - `"sora-2-pro"`

      - `"sora-2-2025-10-06"`

      - `"sora-2-pro-2025-10-06"`

      - `"sora-2-2025-12-08"`

  - `object: "video" (default: "video")`

    The object type, which is always `video`.

    - `"video"`

  - `progress: integer`

    Approximate completion percentage for the generation task.

  - `prompt: string or null`

    The prompt that was used to generate the video.

  - `remixed_from_video_id: string or null`

    Identifier of the source video if this video is a remix.

  - `seconds: string`

    Duration of the generated clip in seconds. For extensions, this is the stitched total duration.

  - `size: VideoSize`

    The resolution of the generated video.

    - `"720x1280"`

    - `"1280x720"`

    - `"1024x1792"`

    - `"1792x1024"`

  - `status: "queued" or "in_progress" or "completed" or "failed"`

    Current lifecycle status of the video job.

    - `"queued"`

    - `"in_progress"`

    - `"completed"`

    - `"failed"`

- `first_id: string or null`

  The ID of the first item in the list.

- `has_more: boolean`

  Whether there are more items available.

- `last_id: string or null`

  The ID of the last item in the list.

- `object: "list" (default: "list")`

  The type of object returned, must be `list`.

  - `"list"`

### Example

```http
curl https://api.openai.com/v1/videos \
    -H "Authorization: Bearer $OPENAI_API_KEY"
```

#### 200 — Example

```json
{
  "data": [
    {
      "id": "id",
      "completed_at": 0,
      "created_at": 0,
      "error": {
        "code": "code",
        "message": "message",
        "headers": {
          "foo": "string"
        },
        "misalignment": {
          "detailed_explanation": "detailed_explanation",
          "error_type": "potentially_unintended_data_transfer",
          "review_target": "review_target",
          "steer": {
            "message": "message"
          }
        },
        "param": "param",
        "type": "type"
      },
      "expires_at": 0,
      "model": "sora-2",
      "object": "video",
      "progress": 0,
      "prompt": "prompt",
      "remixed_from_video_id": "remixed_from_video_id",
      "seconds": "string",
      "size": "720x1280",
      "status": "queued"
    }
  ],
  "first_id": "first_id",
  "has_more": true,
  "last_id": "last_id",
  "object": "list"
}
```

### Example

```http
curl https://api.openai.com/v1/videos \
  -H "Authorization: Bearer $OPENAI_API_KEY"
```

#### Response example

```json
{
  "data": [
    {
      "id": "video_123",
      "object": "video",
      "model": "sora-2",
      "status": "completed",
      "created_at": 1712697600,
      "completed_at": 1712697840,
      "progress": 100,
      "expires_at": null,
      "error": null,
      "prompt": "A calico cat playing a piano on stage",
      "size": "1024x1792",
      "seconds": "8",
      "remixed_from_video_id": null
    }
  ],
  "object": "list",
  "first_id": "video_123",
  "last_id": "video_123",
  "has_more": false
}
```

## Create a video remix

**post** `/videos/{video_id}/remix`

Create a remix of a completed video using a refreshed prompt.

### Path Parameters

- `video_id: string`

  The identifier of the completed video to remix.

### Body Parameters

- `prompt: string`

  Updated text prompt that directs the remix generation.

  `minLength: 1`, `maxLength: 32000`

### Returns

- `Video object { id, completed_at, created_at, 10 more }`

  Structured information describing a generated video job.

  - `id: string`

    Unique identifier for the video job.

  - `completed_at: integer or null`

    Unix timestamp (seconds) for when the job completed, if finished.

    `format: unixtime`

  - `created_at: integer`

    Unix timestamp (seconds) for when the job was created.

    `format: unixtime`

  - `error: VideoCreateError or null`

    Error payload that explains why generation failed, if applicable.

    - `code: string`

      A machine-readable error code that was returned.

    - `message: string`

      A human-readable description of the error that was returned.

    - `headers: optional map[string]`

      The Retry-After and Retry-After-Ms headers returned with the original error, if any.

    - `misalignment: optional object { detailed_explanation, error_type, review_target, steer }`

      - `detailed_explanation: optional string`

        The public explanation for this block.

      - `error_type: optional string or "potentially_unintended_data_transfer" or "potentially_unintended_data_access" or "potentially_unintended_destructive_activity" or "other"`

        An optional classification; clients must accept additional values.

        - `string`

        - `SafetyAlertErrorType = "potentially_unintended_data_transfer" or "potentially_unintended_data_access" or "potentially_unintended_destructive_activity" or "other"`

          An optional classification; clients must accept additional values.

          - `"potentially_unintended_data_transfer"`

          - `"potentially_unintended_data_access"`

          - `"potentially_unintended_destructive_activity"`

          - `"other"`

      - `review_target: optional string or null`

        An opaque target for explicitly continuing this review, or null when unavailable.

        `pattern: ^[A-Za-z0-9._~:-]+$`, `minLength: 1`, `maxLength: 96`

      - `steer: optional object { message }`

        An optional public continuation instruction.

        - `message: string`

          The public continuation instruction.

    - `param: optional string or null`

      The request parameter associated with the error, or null if none was recorded.

    - `type: optional string or null`

      The original error type, or null if it was not recorded.

  - `expires_at: integer or null`

    Unix timestamp (seconds) for when the downloadable assets expire, if set.

    `format: unixtime`

  - `model: VideoModel`

    The video generation model that produced the job.

    - `string`

    - `"sora-2" or "sora-2-pro" or "sora-2-2025-10-06" or 2 more`

      - `"sora-2"`

      - `"sora-2-pro"`

      - `"sora-2-2025-10-06"`

      - `"sora-2-pro-2025-10-06"`

      - `"sora-2-2025-12-08"`

  - `object: "video" (default: "video")`

    The object type, which is always `video`.

    - `"video"`

  - `progress: integer`

    Approximate completion percentage for the generation task.

  - `prompt: string or null`

    The prompt that was used to generate the video.

  - `remixed_from_video_id: string or null`

    Identifier of the source video if this video is a remix.

  - `seconds: string`

    Duration of the generated clip in seconds. For extensions, this is the stitched total duration.

  - `size: VideoSize`

    The resolution of the generated video.

    - `"720x1280"`

    - `"1280x720"`

    - `"1024x1792"`

    - `"1792x1024"`

  - `status: "queued" or "in_progress" or "completed" or "failed"`

    Current lifecycle status of the video job.

    - `"queued"`

    - `"in_progress"`

    - `"completed"`

    - `"failed"`

### Example

```http
curl https://api.openai.com/v1/videos/$VIDEO_ID/remix \
    -H 'Content-Type: application/json' \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -d '{
          "prompt": "x"
        }'
```

#### 200 — Example

```json
{
  "id": "id",
  "completed_at": 0,
  "created_at": 0,
  "error": {
    "code": "code",
    "message": "message",
    "headers": {
      "foo": "string"
    },
    "misalignment": {
      "detailed_explanation": "detailed_explanation",
      "error_type": "potentially_unintended_data_transfer",
      "review_target": "review_target",
      "steer": {
        "message": "message"
      }
    },
    "param": "param",
    "type": "type"
  },
  "expires_at": 0,
  "model": "sora-2",
  "object": "video",
  "progress": 0,
  "prompt": "prompt",
  "remixed_from_video_id": "remixed_from_video_id",
  "seconds": "string",
  "size": "720x1280",
  "status": "queued"
}
```

### Example

```http
curl -X POST https://api.openai.com/v1/videos/video_123/remix \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Extend the scene with the cat taking a bow to the cheering audience"
  }'
```

#### Response example

```json
{
  "id": "video_456",
  "object": "video",
  "model": "sora-2",
  "status": "queued",
  "progress": 0,
  "created_at": 1712698600,
  "size": "720x1280",
  "seconds": "8",
  "remixed_from_video_id": "video_123",
  "prompt": "Extend the scene with the cat taking a bow to the cheering audience",
  "completed_at": null,
  "expires_at": null,
  "error": null
}
```

## Retrieve a video

**get** `/videos/{video_id}`

Fetch the latest metadata for a generated video.

### Path Parameters

- `video_id: string`

  The identifier of the video to retrieve.

### Returns

- `Video object { id, completed_at, created_at, 10 more }`

  Structured information describing a generated video job.

  - `id: string`

    Unique identifier for the video job.

  - `completed_at: integer or null`

    Unix timestamp (seconds) for when the job completed, if finished.

    `format: unixtime`

  - `created_at: integer`

    Unix timestamp (seconds) for when the job was created.

    `format: unixtime`

  - `error: VideoCreateError or null`

    Error payload that explains why generation failed, if applicable.

    - `code: string`

      A machine-readable error code that was returned.

    - `message: string`

      A human-readable description of the error that was returned.

    - `headers: optional map[string]`

      The Retry-After and Retry-After-Ms headers returned with the original error, if any.

    - `misalignment: optional object { detailed_explanation, error_type, review_target, steer }`

      - `detailed_explanation: optional string`

        The public explanation for this block.

      - `error_type: optional string or "potentially_unintended_data_transfer" or "potentially_unintended_data_access" or "potentially_unintended_destructive_activity" or "other"`

        An optional classification; clients must accept additional values.

        - `string`

        - `SafetyAlertErrorType = "potentially_unintended_data_transfer" or "potentially_unintended_data_access" or "potentially_unintended_destructive_activity" or "other"`

          An optional classification; clients must accept additional values.

          - `"potentially_unintended_data_transfer"`

          - `"potentially_unintended_data_access"`

          - `"potentially_unintended_destructive_activity"`

          - `"other"`

      - `review_target: optional string or null`

        An opaque target for explicitly continuing this review, or null when unavailable.

        `pattern: ^[A-Za-z0-9._~:-]+$`, `minLength: 1`, `maxLength: 96`

      - `steer: optional object { message }`

        An optional public continuation instruction.

        - `message: string`

          The public continuation instruction.

    - `param: optional string or null`

      The request parameter associated with the error, or null if none was recorded.

    - `type: optional string or null`

      The original error type, or null if it was not recorded.

  - `expires_at: integer or null`

    Unix timestamp (seconds) for when the downloadable assets expire, if set.

    `format: unixtime`

  - `model: VideoModel`

    The video generation model that produced the job.

    - `string`

    - `"sora-2" or "sora-2-pro" or "sora-2-2025-10-06" or 2 more`

      - `"sora-2"`

      - `"sora-2-pro"`

      - `"sora-2-2025-10-06"`

      - `"sora-2-pro-2025-10-06"`

      - `"sora-2-2025-12-08"`

  - `object: "video" (default: "video")`

    The object type, which is always `video`.

    - `"video"`

  - `progress: integer`

    Approximate completion percentage for the generation task.

  - `prompt: string or null`

    The prompt that was used to generate the video.

  - `remixed_from_video_id: string or null`

    Identifier of the source video if this video is a remix.

  - `seconds: string`

    Duration of the generated clip in seconds. For extensions, this is the stitched total duration.

  - `size: VideoSize`

    The resolution of the generated video.

    - `"720x1280"`

    - `"1280x720"`

    - `"1024x1792"`

    - `"1792x1024"`

  - `status: "queued" or "in_progress" or "completed" or "failed"`

    Current lifecycle status of the video job.

    - `"queued"`

    - `"in_progress"`

    - `"completed"`

    - `"failed"`

### Example

```http
curl https://api.openai.com/v1/videos/$VIDEO_ID \
    -H "Authorization: Bearer $OPENAI_API_KEY"
```

#### 200 — Example

```json
{
  "id": "id",
  "completed_at": 0,
  "created_at": 0,
  "error": {
    "code": "code",
    "message": "message",
    "headers": {
      "foo": "string"
    },
    "misalignment": {
      "detailed_explanation": "detailed_explanation",
      "error_type": "potentially_unintended_data_transfer",
      "review_target": "review_target",
      "steer": {
        "message": "message"
      }
    },
    "param": "param",
    "type": "type"
  },
  "expires_at": 0,
  "model": "sora-2",
  "object": "video",
  "progress": 0,
  "prompt": "prompt",
  "remixed_from_video_id": "remixed_from_video_id",
  "seconds": "string",
  "size": "720x1280",
  "status": "queued"
}
```

## Domain Types

### Image Input Reference Param

- `ImageInputReferenceParam object { file_id, image_url }`

  - `file_id: optional string`

  - `image_url: optional string`

    A fully qualified URL or base64-encoded data URL.

    `maxLength: 20971520`, `format: uri`

### Video

- `Video object { id, completed_at, created_at, 10 more }`

  Structured information describing a generated video job.

  - `id: string`

    Unique identifier for the video job.

  - `completed_at: integer or null`

    Unix timestamp (seconds) for when the job completed, if finished.

    `format: unixtime`

  - `created_at: integer`

    Unix timestamp (seconds) for when the job was created.

    `format: unixtime`

  - `error: VideoCreateError or null`

    Error payload that explains why generation failed, if applicable.

    - `code: string`

      A machine-readable error code that was returned.

    - `message: string`

      A human-readable description of the error that was returned.

    - `headers: optional map[string]`

      The Retry-After and Retry-After-Ms headers returned with the original error, if any.

    - `misalignment: optional object { detailed_explanation, error_type, review_target, steer }`

      - `detailed_explanation: optional string`

        The public explanation for this block.

      - `error_type: optional string or "potentially_unintended_data_transfer" or "potentially_unintended_data_access" or "potentially_unintended_destructive_activity" or "other"`

        An optional classification; clients must accept additional values.

        - `string`

        - `SafetyAlertErrorType = "potentially_unintended_data_transfer" or "potentially_unintended_data_access" or "potentially_unintended_destructive_activity" or "other"`

          An optional classification; clients must accept additional values.

          - `"potentially_unintended_data_transfer"`

          - `"potentially_unintended_data_access"`

          - `"potentially_unintended_destructive_activity"`

          - `"other"`

      - `review_target: optional string or null`

        An opaque target for explicitly continuing this review, or null when unavailable.

        `pattern: ^[A-Za-z0-9._~:-]+$`, `minLength: 1`, `maxLength: 96`

      - `steer: optional object { message }`

        An optional public continuation instruction.

        - `message: string`

          The public continuation instruction.

    - `param: optional string or null`

      The request parameter associated with the error, or null if none was recorded.

    - `type: optional string or null`

      The original error type, or null if it was not recorded.

  - `expires_at: integer or null`

    Unix timestamp (seconds) for when the downloadable assets expire, if set.

    `format: unixtime`

  - `model: VideoModel`

    The video generation model that produced the job.

    - `string`

    - `"sora-2" or "sora-2-pro" or "sora-2-2025-10-06" or 2 more`

      - `"sora-2"`

      - `"sora-2-pro"`

      - `"sora-2-2025-10-06"`

      - `"sora-2-pro-2025-10-06"`

      - `"sora-2-2025-12-08"`

  - `object: "video" (default: "video")`

    The object type, which is always `video`.

    - `"video"`

  - `progress: integer`

    Approximate completion percentage for the generation task.

  - `prompt: string or null`

    The prompt that was used to generate the video.

  - `remixed_from_video_id: string or null`

    Identifier of the source video if this video is a remix.

  - `seconds: string`

    Duration of the generated clip in seconds. For extensions, this is the stitched total duration.

  - `size: VideoSize`

    The resolution of the generated video.

    - `"720x1280"`

    - `"1280x720"`

    - `"1024x1792"`

    - `"1792x1024"`

  - `status: "queued" or "in_progress" or "completed" or "failed"`

    Current lifecycle status of the video job.

    - `"queued"`

    - `"in_progress"`

    - `"completed"`

    - `"failed"`

### Video Create Character Response

- `VideoCreateCharacterResponse object { id, created_at, name }`

  - `id: string or null`

    Identifier for the character creation cameo.

  - `created_at: integer`

    Unix timestamp (in seconds) when the character was created.

    `format: unixtime`

  - `name: string or null`

    Display name for the character.

### Video Create Error

- `VideoCreateError object { code, message, headers, 3 more }`

  An error that occurred while generating the response.

  - `code: string`

    A machine-readable error code that was returned.

  - `message: string`

    A human-readable description of the error that was returned.

  - `headers: optional map[string]`

    The Retry-After and Retry-After-Ms headers returned with the original error, if any.

  - `misalignment: optional object { detailed_explanation, error_type, review_target, steer }`

    - `detailed_explanation: optional string`

      The public explanation for this block.

    - `error_type: optional string or "potentially_unintended_data_transfer" or "potentially_unintended_data_access" or "potentially_unintended_destructive_activity" or "other"`

      An optional classification; clients must accept additional values.

      - `string`

      - `SafetyAlertErrorType = "potentially_unintended_data_transfer" or "potentially_unintended_data_access" or "potentially_unintended_destructive_activity" or "other"`

        An optional classification; clients must accept additional values.

        - `"potentially_unintended_data_transfer"`

        - `"potentially_unintended_data_access"`

        - `"potentially_unintended_destructive_activity"`

        - `"other"`

    - `review_target: optional string or null`

      An opaque target for explicitly continuing this review, or null when unavailable.

      `pattern: ^[A-Za-z0-9._~:-]+$`, `minLength: 1`, `maxLength: 96`

    - `steer: optional object { message }`

      An optional public continuation instruction.

      - `message: string`

        The public continuation instruction.

  - `param: optional string or null`

    The request parameter associated with the error, or null if none was recorded.

  - `type: optional string or null`

    The original error type, or null if it was not recorded.

### Video Delete Response

- `VideoDeleteResponse object { id, deleted, object }`

  Confirmation payload returned after deleting a video.

  - `id: string`

    Identifier of the deleted video.

  - `deleted: boolean`

    Indicates that the video resource was deleted.

  - `object: "video.deleted" (default: "video.deleted")`

    The object type that signals the deletion response.

    - `"video.deleted"`

### Video Get Character Response

- `VideoGetCharacterResponse object { id, created_at, name }`

  - `id: string or null`

    Identifier for the character creation cameo.

  - `created_at: integer`

    Unix timestamp (in seconds) when the character was created.

    `format: unixtime`

  - `name: string or null`

    Display name for the character.

### Video Model

- `VideoModel = string or "sora-2" or "sora-2-pro" or "sora-2-2025-10-06" or 2 more`

  - `string`

  - `"sora-2" or "sora-2-pro" or "sora-2-2025-10-06" or 2 more`

    - `"sora-2"`

    - `"sora-2-pro"`

    - `"sora-2-2025-10-06"`

    - `"sora-2-pro-2025-10-06"`

    - `"sora-2-2025-12-08"`

### Video Seconds

- `VideoSeconds = "4" or "8" or "12"`

  - `"4"`

  - `"8"`

  - `"12"`

### Video Size

- `VideoSize = "720x1280" or "1280x720" or "1024x1792" or "1792x1024"`

  - `"720x1280"`

  - `"1280x720"`

  - `"1024x1792"`

  - `"1792x1024"`
