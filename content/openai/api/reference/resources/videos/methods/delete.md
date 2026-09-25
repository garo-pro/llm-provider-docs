> For the complete documentation index, see [llms.txt](/llms.txt). Markdown versions of documentation pages are available by appending `.md` to the page URL.

> The Sora 2 models and Videos API were shut down on September 24, 2026 and are no longer available. No one-to-one replacement API is available. This page is retained for historical reference. See the [deprecation notice](https://developers.openai.com/api/docs/deprecations#2026-03-24-sora-2-video-generation-models-and-videos-api).

## Delete a video

**delete** `/videos/{video_id}`

Permanently delete a completed or failed video and its stored assets.

### Path Parameters

- `video_id: string`

### Returns

- `id: string`

  Identifier of the deleted video.

- `deleted: boolean`

  Indicates that the video resource was deleted.

- `object: "video.deleted"`

  The object type that signals the deletion response.

  - `"video.deleted"`

### Example

```http
curl https://api.openai.com/v1/videos/$VIDEO_ID \
    -X DELETE \
    -H "Authorization: Bearer $OPENAI_API_KEY"
```

#### Response

```json
{
  "id": "id",
  "deleted": true,
  "object": "video.deleted"
}
```
