# Agent Guidelines for Ansible Dotfiles

This repository manages mutable machine configuration using Ansible. Each program configuration is managed by a dedicated role.

## Project Overview

**Type**: Ansible playbook for dotfiles management  
**Main Playbook**: `site.yaml`  
**Roles**: bat, btop, dunst, git, gtk, hyprland, k9s, kitty, qt, rofi, theme, waybar, yazi, zsh

## Build/Run Commands

### Running the Full Playbook
```bash
ansible-playbook site.yaml
```

### Running Specific Roles
```bash
# Single role
ansible-playbook site.yaml --tags rofi

# Multiple roles
ansible-playbook site.yaml --tags "hyprland,waybar,dunst"
```

### Running Specific Task Categories
```bash
# Only config tasks (no systemd)
ansible-playbook site.yaml --tags config

# Only systemd tasks
ansible-playbook site.yaml --tags systemd
```

### Dry Run / Check Mode
```bash
# See what would change without applying
ansible-playbook site.yaml --check

# With detailed diff output
ansible-playbook site.yaml --check --diff
```

### Listing Available Tasks and Tags
```bash
# List all tasks
ansible-playbook site.yaml --list-tasks

# List all available tags
ansible-playbook site.yaml --list-tags
```

### Syntax Checking
```bash
# Validate playbook syntax
ansible-playbook site.yaml --syntax-check
```

### Linting
```bash
# Lint all playbooks and roles
ansible-lint

# Lint specific files
ansible-lint site.yaml
ansible-lint roles/hyprland/
```

## Repository Structure

```
.
├── site.yaml                    # Main playbook
├── roles/                       # All role definitions
│   ├── theme/                   # Special role: loads theme data for other roles
│   │   ├── defaults/main.yaml   # Default variables
│   │   ├── tasks/main.yaml      # Task definitions
│   │   ├── files/themes/        # Theme assets (base16.yaml, polarity.txt, wp.jpg)
│   │   └── README.md            # Role documentation
│   └── {role_name}/
│       ├── defaults/main.yaml   # Default variables (source directories)
│       ├── tasks/main.yaml      # Task definitions
│       ├── templates/           # Jinja2 templates (*.j2)
│       └── files/               # Static files
```

## Code Style Guidelines

### YAML Formatting

- **Indentation**: 2 spaces (no tabs)
- **String quoting**: Quote strings containing variables or special characters
- **Booleans**: Use `true`/`false` (lowercase)
- **File permissions**: Always specify as quoted octal strings (e.g., `"0700"`, `"0600"`)

### Ansible Task Structure

**Standard task pattern for all roles:**

```yaml
- name: Create source directory
  ansible.builtin.file:
    path: "{{ role_source_dir }}"
    state: directory
    owner: "{{ ansible_facts['user_id'] }}"
    group: "{{ ansible_facts['user_id'] }}"
    mode: "0700"
  become: true
  tags: [role_name, config]
```

**Template rendering pattern:**

```yaml
- name: Render template configurations
  ansible.builtin.template:
    src: "{{ item }}.j2"
    dest: "{{ role_source_dir }}/{{ item }}"
    owner: "{{ ansible_facts['user_id'] }}"
    group: "{{ ansible_facts['user_id'] }}"
    mode: "0600"
  loop:
    - config_file_1
    - config_file_2
  tags: [role_name, config]
```

**File copying pattern:**

```yaml
- name: Copy file configurations
  ansible.builtin.copy:
    src: "{{ item }}"
    dest: "{{ role_source_dir }}/{{ item }}"
    owner: "{{ ansible_facts['user_id'] }}"
    group: "{{ ansible_facts['user_id'] }}"
    mode: "0600"
  loop:
    - static_file_1
  tags: [role_name, config]
```

### Naming Conventions

- **Variables**: Use snake_case with role name prefix (e.g., `dunst_source_dir`, `theme_name`)
- **Task names**: Use descriptive sentence case (e.g., "Create source directory", "Render template configurations")
- **Tags**: Use lowercase, role-specific tags; always include role name and category (e.g., `[waybar, config]`, `[dunst, systemd]`)
- **Files**: Use `.yaml` extension (not `.yml`)
- **Templates**: Always use `.j2` extension for Jinja2 templates

### Module Usage

- **Always use FQCN**: Use fully qualified collection names (e.g., `ansible.builtin.file`, not just `file`)
- **Preferred modules**:
  - File operations: `ansible.builtin.file`, `ansible.builtin.copy`, `ansible.builtin.template`
  - Systemd: `ansible.builtin.systemd_service` (always specify `scope: user` for user services)
  - Variables: `ansible.builtin.set_fact`, `ansible.builtin.include_vars`
  - Downloads: `ansible.builtin.get_url`, `ansible.builtin.uri`

### Variables and Facts

- **Access user info**: Use `ansible_facts['user_id']` and `ansible_facts['user_dir']`
- **Theme data**: Access via `theme_data.colors.base00` through `base0F`, `theme_data.font.style.*`, `theme_data.wallpaper_path`
- **Default variables**: Define in `roles/{role}/defaults/main.yaml` with role prefix
- **Changed detection**: Use `changed_when: false` for read-only operations

### Jinja2 Templates

- **Variable interpolation**: Use `{{ variable_name }}` syntax
- **Color references**: Use `#{{ theme_data.colors.baseXX }}` for hex colors (no leading # in base16.yaml values)
- **Font references**: 
  - `{{ theme_data.font.style.mono }}` - Monospace font
  - `{{ theme_data.font.style.serif }}` - Serif font
  - `{{ theme_data.font.style.sansSerif }}` - Sans-serif font
  - `{{ theme_data.font.size.terminal }}`, `.apps`, `.popups`, `.desktop` - Font sizes
- **Defaults**: Use `| default(fallback_value)` for optional theme variables

### Tags Strategy

- **Primary tags**: Always include role name (e.g., `[hyprland]`)
- **Category tags**: Add task category (`config`, `systemd`, etc.)
- **Multiple tags**: Group related functionality with shared tags

### Systemd Service Management

```yaml
- name: Enable service_name service
  ansible.builtin.systemd_service:
    name: service_name.service
    enabled: true
    state: started  # or 'restarted' to force reload
    scope: user     # Always use 'user' scope for user services
  tags: [role_name, systemd]
```

**Important**: Never use `daemon_reload: true` in service tasks (breaks idempotency)

### File Permissions

- **Directories**: `mode: "0700"`
- **Config files**: `mode: "0600"`
- **Executables**: `mode: "0755"`
- **Always set**: `owner` and `group` to `ansible_facts['user_id']`

## Common Patterns

### Creating a New Role

1. Create role directory: `roles/{role_name}/`
2. Add subdirectories: `defaults/`, `tasks/`, `templates/`, `files/`
3. Define default variables in `defaults/main.yaml`
4. Create tasks in `tasks/main.yaml` following standard patterns above
5. Add role to `site.yaml` roles list
6. Use appropriate tags for selective execution

### Theme Integration

The `theme` role must run first. It sets `theme_data` fact containing:
- `theme_data.colors.base00` through `base0F` (base16 color scheme)
- `theme_data.polarity` ("dark" or "light")
- `theme_data.wallpaper_path`
- `theme_data.font.style.{mono,serif,sansSerif}`
- `theme_data.font.size.{terminal,apps,popups,desktop}`
- `theme_data.kb_layout` and `theme_data.kb_variant`

All other roles consume this data for consistent theming.

### Error Handling

- Use `ansible.builtin.assert` for validation with clear `fail_msg`
- Register operation results when needed: `register: variable_name`
- Use `changed_when: false` for read-only tasks
- Validate checksums for downloads

## Testing

This is a configuration management repository without traditional unit tests. Testing is done through:

1. **Syntax validation**: `ansible-playbook site.yaml --syntax-check`
2. **Dry runs**: `ansible-playbook site.yaml --check --diff`
3. **Linting**: `ansible-lint`
4. **Manual verification**: Apply to test system and verify configurations

## Best Practices

- Always create source directories before copying/templating files
- Use `become: true` when creating directories in user home
- Prefer templates over static files for theme-dependent configs
- Keep static content in `files/`, dynamic content in `templates/`
- Document role variables in role README.md (see `roles/theme/README.md`)
- Use loops for multiple similar tasks
- Tag tasks appropriately for selective execution
