# Theme

Reads theme configuration files and sets facts for use by other roles.

## Role Variables

### theme_name
The name of the theme to load. Must match a directory name in `files/themes/`.

Available themes:
- everforest
- nord
- uwunicorn

Default: `everforest`

## Facts Set

This role sets the following facts:

- `theme.polarity`: Theme polarity (e.g., "dark", "light")
- `theme.colors`: Dictionary containing base16 color scheme:
  - `scheme`: Theme name
  - `author`: Theme author
  - `base00` - `base0F`: Hex color values
- `theme.wallpaper_path`: Absolute path to the theme wallpaper image

## Example Usage

```yaml
- hosts: servers
  roles:
    - role: theme
      theme_name: nord
```

After this role runs, colors can be accessed via `theme_data.colors.base00`, etc.

