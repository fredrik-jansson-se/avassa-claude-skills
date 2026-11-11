# avassa-claude-skills

Claude Code skills for generating Avassa payloads: application specs, application deployments, and Strongbox vaults/secrets.

## Contents

| Plugin   | Skill             | Description                                                                     |
|----------|-------------------|---------------------------------------------------------------------------------|
| `avassa` | `avassa-app-spec` | Writes Avassa application specs, deployments and Strongbox vaults/secrets (YAML) |

## Output

The skill writes payloads in `supctl patch` format (each document carries `x-path` and `x-operation` headers). By default all payloads (TLS CA, vault, secrets, application, deployment) go in a single multi-document YAML file, applied with:

```sh
supctl patch < myapp.yaml
```

To get one numbered file per payload instead (as in the [Avassa application examples](https://gitlab.com/avassa-public/application-examples)), ask for it, e.g. "put each resource in a separate file". You can also ask Claude to split or merge existing output.

## Installation

This repository is a Claude Code plugin marketplace (`avassa-skills`) containing the `avassa` plugin.

From inside Claude Code:

```
/plugin marketplace add fredrik-jansson-se/avassa-claude-skills
/plugin install avassa@avassa-skills
```

Or from the shell:

```sh
claude plugin marketplace add fredrik-jansson-se/avassa-claude-skills
claude plugin install avassa@avassa-skills
```

Restart Claude Code (or run `/reload-plugins` if available) after installing. The skill then loads automatically when you ask Claude to write Avassa application specs, deployments or secrets.

### From a local clone

```sh
git clone git@github.com:fredrik-jansson-se/avassa-claude-skills.git
```

```
/plugin marketplace add /path/to/avassa-claude-skills
/plugin install avassa@avassa-skills
```

### Manual install (without the plugin system)

Copy the skill directory into your personal skills folder:

```sh
mkdir -p ~/.claude/skills
cp -r plugins/avassa/skills/avassa-app-spec ~/.claude/skills/
```

## Updating

Refresh the marketplace to pull the latest version from GitHub, then update the plugin:

```
/plugin marketplace update avassa-skills
```

```sh
claude plugin update avassa@avassa-skills
```

Restart Claude Code afterwards to load the new version.

If you installed manually, pull the repository and copy the skill directory again:

```sh
git pull
cp -r plugins/avassa/skills/avassa-app-spec ~/.claude/skills/
```

### For maintainers

Bump `version` in `plugins/avassa/.claude-plugin/plugin.json` when releasing changes, so installed copies detect the new version.

## Uninstalling

```
/plugin uninstall avassa@avassa-skills
/plugin marketplace remove avassa-skills
```

## License

See [LICENSE](LICENSE).
