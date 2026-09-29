# excalidashapi-mcp

An MCP server for working with drawings in [ExcaliDash](https://github.com/ZimengXiong/ExcaliDash). An AI agent can create a drawing, edit shapes and text, connect elements, add images, and organize drawings into collections. ExcaliDash stores the drawings; changes remain visible in its editor.

The server runs over stdio. Your MCP client starts it as a child process and supplies an ExcaliDash user API key. You do not need to host another MCP endpoint, install a database, or give the server S3 credentials.

## Connect a client

You need a running ExcaliDash instance with the Drawing Agent API and an account API key. In ExcaliDash, create the key under **Profile → API keys** with `drawings:read`, `drawings:write`, `collections:read`, and `collections:write`. The drawing tools need only the two drawing scopes; collection tools need the collection scopes too.

Use the public **frontend URL** of your ExcaliDash instance. The API client appends `/api` itself; do not point it at a backend service address. Your MCP client needs Node.js 20.6 or newer for the npm package and the development command below, or Docker for the container image.

### npm

Add a stdio server to a client that uses the `mcpServers` configuration format:

```json
{
  "mcpServers": {
    "excalidash": {
      "command": "npx",
      "args": ["-y", "excalidashapi-mcp@0.1.0"],
      "env": {
        "EXCALIDASH_URL": "https://draw.example.com",
        "EXCALIDASH_API_KEY": "YOUR_ACCOUNT_KEY"
      }
    }
  }
}
```

Replace the URL and key. The package name is `excalidashapi-mcp`; `excalidash-mcp` without `api` is a different npm package. The version is pinned so a client restart cannot silently install a newer release.

### Container image

For a client that launches Docker, use the image from GitHub Container Registry:

```json
{
  "mcpServers": {
    "excalidash": {
      "command": "docker",
      "args": [
        "run", "--rm", "-i",
        "--env", "EXCALIDASH_URL",
        "--env", "EXCALIDASH_API_KEY",
        "ghcr.io/latikdesu/excalidash-mcp:0.1.0"
      ],
      "env": {
        "EXCALIDASH_URL": "https://draw.example.com",
        "EXCALIDASH_API_KEY": "YOUR_ACCOUNT_KEY"
      }
    }
  }
}
```

The image runs the stdio server by default. Keep `-i` for the protocol stream; do not add `-t` or publish a port. For long-lived configurations, you can replace the version tag with the image digest shown by GHCR after publishing.

Client configuration formats differ. If your client does not use `mcpServers`, supply the same command, arguments, and two environment variables in its server settings. Protect the config file if it contains the key, or use the client's secret store. Neither the npm package nor the image contains your credentials.

## Tools

The server exposes 17 tools. Drawing selection belongs to one MCP session; another session will not inherit it.

| Tool | What it does |
|---|---|
| `list_drawings` | Lists accessible drawings. Supports `search`, `limit` (1–100), and `offset`. |
| `create_drawing` | Creates an empty Excalidraw drawing. Selects it by default unless `select` is false. |
| `select_drawing` | Reads and selects a drawing for tools that accept an optional `drawingId`. A failed selection leaves the previous choice intact. |
| `get_selected_drawing` | Returns the drawing selected in this MCP session. |
| `get_drawing_summary` | Reads a compact structural summary of a drawing. |
| `inspect_drawing_element` | Returns one element's properties and its bound children. |
| `apply_drawing_ops` | Atomically applies up to 50 semantic operations in a single backend request. |
| `rename_drawing` | Renames a drawing by its explicit ID. |
| `delete_drawing` | Permanently deletes a drawing by its explicit ID. It does not move it to Trash. |
| `get_drawing` | Returns the full stored scene, including `elements`, `appState`, `files`, and the current `version`. |
| `find_elements` | Finds non-deleted elements by exact ID/type and a literal, case-sensitive text substring; filters are combined with AND. Results are paginated, up to 100 per page. |
| `list_collections` | Lists owned and shared collections with the ownership metadata provided by ExcaliDash. |
| `create_collection` | Creates a collection owned by the key's user. |
| `rename_collection` | Renames an owned collection. Trash cannot be renamed. |
| `delete_collection` | Deletes an owned collection and its shares. Its drawings are kept, without a collection. Trash cannot be deleted. |
| `move_drawing_to_collection` | Moves a drawing to an existing collection, or uses `collectionId: null` to leave it unorganized. |
| `add_image` | Adds a PNG or JPEG image from base64 and places it at the supplied coordinates. Decoded input is limited to 5 MiB. |

### Drawing operations

`apply_drawing_ops` supports `add_shape`, `connect`, `set_text`, `set_style`, `move`, `delete`, `import_elements`, `layout`, and `revert_to_snapshot`. A `layout` operation accepts graph nodes and edges; supported directions are `TB`, `BT`, `LR`, and `RL`. A graph can have up to 200 nodes and 400 edges, with a batch limit of 300 nodes and 600 edges.

When adding shapes, you can give them short `ref` names. The response maps these names to element IDs. Use the names in a **second** `apply_drawing_ops` call to connect or edit those elements. ExcaliDash assigns IDs when it creates the shapes, so a request that creates a `ref` and refers to it later in the *same* batch is rejected before writing. Each call is atomic on its own; the two calls are not one transaction. Aliases are scoped to the current drawing and MCP session and disappear on restart.

`revert_to_snapshot` restores elements from a version you already know. It is not a history browser or a full restoration of files and app state. Reverted elements may remain in the stored scene with `isDeleted: true`.

### Images and stored data

`add_image` accepts canonical base64 without a `data:` prefix, `mimeType` (`image/png` or `image/jpeg`), and `x`, `y`, `width`, `height`. The server checks the encoded data and image signature before sending it to ExcaliDash. A version conflict is returned as an error; the tool does not retry over another editor's changes.

`get_drawing` reads the stored scene. Its `files` entries may contain private ExcaliDash references rather than embedded image bytes. To make a portable `.excalidraw` file, use the export button in ExcaliDash itself.

## Development

```sh
npm ci
npm test
```

For a direct stdio run, copy [`.env.example`](.env.example) to `.env`, set `EXCALIDASH_URL` and `EXCALIDASH_API_KEY`, restrict the file to mode `0600`, and run `node --env-file=.env dist/index.js` after `npm run build`. Keep browser-login passwords out of this file.

Streamable HTTP is available for development and transport testing: `node --env-file=.env dist/http.js` serves `/mcp` on `127.0.0.1:8080` by default. `/health` checks the process; `/ready` makes a bounded, authenticated read request to ExcaliDash. The Host and Origin allowlists are configurable through the variables in `.env.example`. They are not a substitute for client authentication on an exposed network endpoint. `docker compose up --build -d` runs this HTTP mode with a loopback-only published port.

`npm test` does not change drawings on a running ExcaliDash instance. Run `npm audit` to check dependencies.

Licensed under [MIT](LICENSE).
