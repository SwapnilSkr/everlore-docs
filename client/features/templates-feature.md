# Templates Feature

The Templates feature allows players to browse published world templates and create new game instances from them.

---

## File Structure

```
features/templates/
├── data/
│   └── template_repository.dart
├── presentation/
│   ├── browse_screen.dart
│   └── template_detail_screen.dart
└── state/                (none — uses setState directly)
```

> **Note:** This feature does not use Cubit/BLoC. Both screens manage state via `StatefulWidget` + `setState()`. This is a simpler approach since the screens are self-contained and don't need reactive rebuilds from external state changes.

---

## BrowseTemplatesScreen

**File:** `lib/features/templates/presentation/browse_screen.dart`

Route: `/templates`

A searchable list of published world templates.

### Local State

| Field | Type | Description |
|-------|------|-------------|
| `_templates` | `List<WorldTemplate>` | Loaded template list |
| `_isLoading` | `bool` | Loading indicator |
| `_error` | `String?` | Error message |
| `_searchController` | `TextEditingController` | Search input controller |

### UI Layout

```
┌────────────────────────────────────┐
│ AppBar: "Browse Worlds"            │
├────────────────────────────────────┤
│ [🔍 Search worlds...        ✕]     │  ← Search text field
├────────────────────────────────────┤
│                                    │
│ _TemplateCard                      │  ← Repeated for each template
│ _TemplateCard                      │
│ _TemplateCard                      │
│                                    │
└────────────────────────────────────┘
```

### Search Behavior
- Typing and pressing Enter triggers `_loadTemplates(search: val)`
- Clear button (`✕`) resets search and reloads all templates
- No debounced search — only triggers on submit

### UI States
| State | UI |
|-------|-----|
| Loading | Centered purple `CircularProgressIndicator` |
| Error | Centered red error text |
| Empty results | "No worlds found" centered text |
| Loaded | `ListView` of `_TemplateCard` widgets |

---

## _TemplateCard (Private Widget)

A card displaying summary information about a `WorldTemplate`.

### Layout

```
┌────────────────────────────────────────┐
│ 🧠 World Title              [NSFW]     │  ← Icon (sentient/stories) + title + NSFW badge
│                                        │
│ A dark fantasy world where...          │  ← Description (3 lines max)
│                                        │
│ [combat] [exploration] [dialogue] [...]│  ← Scene tag chips (max 4)
└────────────────────────────────────────┘
```

### Display Logic
- **Icon:** `Icons.psychology` (purple) for sentient templates, `Icons.auto_stories` (blue) for standard
- **NSFW badge:** Red "NSFW" chip shown if `template.isNsfwCapable` is true
- **Scene tags:** Up to 4 tags rendered as `Chip` widgets

---

## TemplateDetailScreen

**File:** `lib/features/templates/presentation/template_detail_screen.dart`

Route: `/templates/:templateId`

Detailed view of a single template with stats preview and "Enter This World" button.

### Local State

| Field | Type | Description |
|-------|------|-------------|
| `_template` | `WorldTemplate?` | Loaded template data |
| `_isLoading` | `bool` | Loading indicator |
| `_isCreating` | `bool` | Instance creation in progress |

### Initialization
On `initState`, calls `_load()` which fetches the template via `TemplateRepository.getById()`.

### UI Layout

```
┌────────────────────────────────────┐
│ AppBar: Template title             │
├────────────────────────────────────┤
│ 🧠  World Title                    │
│     Sentient World / GM World      │
│                                    │
│ Description text...                │
│                                    │
│ World Stats                        │
│ Health ████████░░ 80/100           │  ← StatBar for each stat
│   Your physical health             │  ← Stat description
│ Mana   ████░░░░░░ 40/100           │
│   Your magical energy              │
│                                    │
│ Scene Types                        │
│ [combat] [exploration] [dialogue]  │  ← Scene tag chips
│                                    │
├────────────────────────────────────┤
│     [ Enter This World ]           │  ← Bottom button (pinned)
└────────────────────────────────────┘
```

### Stats Preview
Each entry in `baseStatsTemplate` is rendered as:
1. `StatBar` with label, default value, and max
2. Description text below the bar

### Instance Creation
When "Enter This World" is pressed:
1. Sets `_isCreating = true` (disables button, shows spinner)
2. Calls `HomeRepository.createInstance(templateId)`
3. On success: `context.go('/play/${instance.id}')` (replaces navigation stack)
4. On failure: Shows `SnackBar` with error message

### UI States
| State | UI |
|-------|-----|
| Loading | Centered purple `CircularProgressIndicator` |
| Not found | "Template not found" centered text |
| Loaded | Full detail view with stats and tags |
| Creating | Bottom button shows spinning indicator |

---

## TemplateRepository

**File:** `lib/features/templates/data/template_repository.dart`

### Methods

| Method | API Call | Returns | Description |
|--------|----------|---------|-------------|
| `listPublished({page, limit, search})` | `GET /templates?page=N&limit=N&search=Q` | `Map` with `templates`, `total` | Searchable paginated template list |
| `getById(id)` | `GET /templates/:id` | `WorldTemplate` | Single template details |

### Response Format

**List Response:**
```json
{
  "templates": [
    { "_id": "...", "title": "...", "description": "...", ... }
  ],
  "total": 42
}
```

**Get By ID Response:**
```json
{
  "_id": "...",
  "title": "...",
  "base_stats_template": {
    "health": { "default": 80, "min": 0, "max": 100, "description": "Physical health" }
  }
}
```
