# The APEX Developer Reinvented

This repo is to share information provided through my blog series at [https://dgielis.com/the-apex-developer-reinvented](https://dgielis.com/the-apex-developer-reinvented)
and presentations that I've been given at APEX World, APEX Alpe Adria and KScope in 2026.

## Presentation abstract

AI can build an app in minutes. Give it a prompt, and you get a frontend, backend, and database… all wired together.
Pretty impressive.
But would you deploy that in a real enterprise? Probably not.
Because when it matters, we still care about security, performance, data integrity, and maintainability. And that is exactly where Oracle APEX and the Oracle Database shine.
In this session, I will show how AI is changing the way we build with APEX, and how our role as developers is evolving with it. We are no longer just creating pages and writing code. We become designers, orchestrators, and validators of what AI produces.
Using a World Cup prediction app as a running example, I will go from idea to a working application with AI in the loop. From data model, to PL/SQL API, to a full APEX app. You will see how AI agents can accelerate development, and where human expertise still makes the difference.
This is not about replacing developers. It is about becoming a 10x developer with AI.
If you work with Oracle APEX, or are curious about what is coming next, this session will give you a real, practical look at the future of development.
There is a lot to explore 😁

## Presentation download

Provided after KScope (JUL 2026).

## Supporting files

### APEX Bootstrap 

Setup an entire Oracle APEX environment locally from a single executable.

#### Stack

- Oracle Database 26ai, ORDS 25.4, and APEX 24.2 in Docker for `DEV` and `TEST` by default, with optional `PROD`
- SQLcl Projects for database source control, staging, release, and deployment
- GitLab for source control, merge requests, and issue tracking
- Jenkins for CI/CD orchestration, with optional approval-gated `PROD` releases
- utPLSQL for PL/SQL tests
- Playwright for browser and smoke regression tests

#### Layout

- `database/project/app/`: SQLcl Project root template
- `database/source/`: hand-authored schema source files
- `database/deploy/`: bootstrap and helper deployment SQL
- `apex/f1000/`: APEX split export placeholder for app `1000`
- `ords/modules/`: ORDS module definitions
- `tests/utplsql/`: utPLSQL suites
- `tests/playwright/`: browser tests and config
- `docker/`: local environment compose files
- `scripts/`: helper scripts used by Jenkins and local development

#### Branch and environment model

- `feature/*`: built directly against shared `DEV`
- `develop`: integration branch with validation-only CI
- `test`: auto-deploys to `TEST`
- `main`: optional approval-gated deploy to `PROD` when bootstrap ran with `--include-production`

#### Download

https://drive.google.com/drive/folders/1UOoht6u10UpiGaxfasj1z8sUfXQEua2A?usp=sharing

#### Usage

Run the executable for your platform

  apex-bootstrap init -p NAME [-r DIR] [--force] [--include-production]
                                                      Full platform bootstrap
  apex-bootstrap add -p NAME [--app-id ID] [--include-production]
                                                      Add project to running infra
  apex-bootstrap uninstall [--yes]                    Remove bootstrap-owned state
  apex-bootstrap docker-clean [--yes]                 Nuke all Docker state (except images)
  apex-bootstrap -p NAME                                Backwards compat → init
  apex-bootstrap vk                                   Install Vibe Kanban Board dependencies

Options:
  -p, --project-name NAME   Project name (default: APEX App)
  -r, --repo-parent DIR     Parent directory for <slug>-app (default: cwd)
  -R, --repo-dir DIR        Exact directory to create/use
  -P, --include-production  Also provision and wire PROD
  --force                   Allow extraction into non-empty directory
  --skip-bootstrap          Extract only; skip provisioning
  -y, --yes                 Confirm destructive uninstall
  --app-id ID               APEX app ID for add mode (auto-detected if omitted)
  -h, --help                Show this help

Commands:
  init    Set up full local platform (DB, ORDS, GitLab, Jenkins, CI/CD)
  add     Add a new project to already-running infrastructure
  uninstall
          Remove bootstrap-owned Docker resources, tracked repos, installed deps,
          and Homebrew if it was installed by apex-bootstrap
  docker-clean
          Stop and remove ALL Docker containers, volumes, networks, and build
          cache. Images are kept. This is a full Docker reset.
  vk      Install the dependencies needed to run Vibe Kanban Board:
          fnm, Node.js LTS, pnpm, Rust, cargo-watch, sqlx-cli,
          Claude Code, and Codex, then clone vibe-kanban into
          ./vibe-kanban or $VK_REPO_DIR using GH_TOKEN/GITHUB_TOKEN or SSH
