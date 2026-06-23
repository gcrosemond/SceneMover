# SceneMover

A [Stash](https://github.com/stashapp/stash) plugin that plans, previews, moves, and renames scene files from scene metadata.

It is built around two concepts:

- `roots`: destination anchors, each with a path template
- `rules`: ordered matching rules that choose which root a scene should use

## Features

- Rule-based routing across multiple libraries or sub-libraries
- Destination templates with scene metadata tokens
- Optional tokens with `{_token}` syntax
- Conditional template blocks with `[[ ... ]]` syntax
- Preview mode before applying moves
- Single-scene move from scene-card badges
- Bulk move for selected scenes
- Misplaced-only filter on scene card lists
- Auto-resolution for name collisions
- Same-folder rename handling
- Empty source folder cleanup
- Windows path-length mitigation by trimming `{title}` then `{performers}`
- Relative templates anchored on the matched root path

## Installation

1. Copy the plugin directory into your Stash plugins folder.
2. Reload plugins in Stash.
3. Open SceneMover from the Stash UI and configure roots and rules.

## Configuration Model

### Roots

A root defines:

- a label
- a root path
- a template
- whether it is the default fallback

If a template is relative, it is anchored on the matched root path.

Example:

```text
Root path: /Volumes/Media/Scenes
Template:  __sorted/{studio} - {title} [{_date}].{ext}
```

This renders under:

```text
/Volumes/Media/Scenes/__sorted/...
```

If a template is absolute, it is used as-is.

### Rules

Rules are evaluated top to bottom. The first matching rule selects the root.

Current rule conditions:

- `performer_is_favourite`
- `studio_equals`
- `studio_contains`
- `tag_equals`
- `performer_equals`
- `file_under_dir`
- `has_group`

`file_under_dir` matches against the current file directory, recursively.

If no rule matches, the default root is used.

## Template Syntax

Templates can include both folders and filenames. Use `/` or `\` in the template to create subdirectories; the final component is treated as the filename.

Example:

```text
{studioFirstLetter}/{studio}/{title}[[ - {_performers}]][[ [{_date}]]].{ext}
```

### Tokens

| Token | Value |
|---|---|
| `{studio}` | Studio name |
| `{studioFirstLetter}` | First letter of studio name |
| `{studioInitial}` | Alias for `studioFirstLetter` |
| `{date}` | Full date in `YYYY-MM-DD` form |
| `{yyyy-MM-dd}` | Full date in `YYYY-MM-DD` form |
| `{yyyy-MM}` | Year and month |
| `{yyyy}` | Year |
| `{MM-dd}` | Month and day |
| `{title}` | Scene title |
| `{performers}` | All performer names, comma-separated |
| `{favoritedPerformer}` | First favourited performer, or first performer |
| `{favoritedPerformerInitial}` | First letter of `{favoritedPerformer}` |
| `{favoritedPerformerFirstLetter}` | Alias for `{favoritedPerformerInitial}` |
| `{codec}` | Video codec |
| `{height}` | Vertical resolution |
| `{ext}` | File extension without the dot |
| `{scene_id}` | Scene code if present and short enough |

### Optional Tokens

Prefix a token with `_` to make it optional:

```text
{title} - {_performers} [{_date}].{ext}
```

Behavior:

- if the token has a value, it is expanded normally
- if the token is missing, it expands to an empty string
- the renderer then performs light cleanup of leftover repeated spaces and dots

This is useful when missing metadata should not block the move.

### Conditional Blocks

Wrap a larger section in `[[ ... ]]` when the whole section should disappear unless at least one token inside has a value.

Example:

```text
{title}[[ - {_performers}]][[ [{_date}]]].{ext}
```

Results:

- with performers and date: `Title - Name [2024-01-02].mp4`
- with performers only: `Title - Name.mp4`
- with date only: `Title [2024-01-02].mp4`
- with neither: `Title.mp4`

This solves the common `foo []` problem that plain optional tokens cannot solve by themselves.

### Template Recipes

#### Flat filename

Template:

```text
{studio} - {title}[[ - {_performers}]][[ [{_date}]]].{ext}
```

Example result:

```text
Brazzers - Example Scene - Jane Doe [2024-01-02].mp4
```

Use this when you want everything in one folder and only the filename to vary.

#### Per-studio subfolders

Template:

```text
{studioFirstLetter}/{studio}/{title}[[ - {_performers}]][[ [{_date}]]].{ext}
```

Example result:

```text
B/Brazzers/Example Scene - Jane Doe [2024-01-02].mp4
```

Use this when you want a shallow alphabetical split before the studio folder.

#### Optional performer and date blocks

Template:

```text
{title}[[ - {_performers}]][[ [{_date}]]].{ext}
```

Example results:

```text
Example Scene - Jane Doe [2024-01-02].mp4
Example Scene - Jane Doe.mp4
Example Scene [2024-01-02].mp4
Example Scene.mp4
```

Use this when `performers` and `date` are nice to have but should not leave visual artifacts when missing.

## Validation Behavior

Template validation distinguishes between required and optional metadata:

- required tokens must have values or the scene is skipped
- `_optional` tokens do not cause a skip
- tokens inside `[[ ... ]]` are treated as optional for validation

In practice this means you can make `date` or `performers` optional without weakening validation for fields you actually depend on.

## Preview and Apply

Preview computes the destination path for each file and shows whether it will:

- move to another folder
- be renamed in place
- be skipped

Apply performs the planned moves and:

- creates destination folders as needed
- auto-resolves filename collisions
- uses the matched root as the anchor for relative templates
- cleans up empty source folders after successful moves

## Misplaced Overlay

On scene card views, SceneMover can mark misplaced scenes directly in the UI:

- `Wrong drive`
- `Wrong folder`
- `Filename mismatch`

The toolbar button can toggle a misplaced-only filter for that view.

## Notes and Limitations

- Destination folders must remain within a valid Stash library path when Stash itself performs the move.
- Auto-resolution is intentionally kept for filename collisions.
- Path rendering is normalized across Windows and Unix-style separators.
- Very long paths are shortened by trimming filename components, not by changing folder routing.
