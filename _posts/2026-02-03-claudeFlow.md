---
title: My AI Workflow
date: 2026-02-03 09:10 -0600
categories: [Computers, Nix]
tags: [linux, nix, computer, ai]
description: A guide on what nix is from a long time nix user
---

# My Claude Workflow

I'll preface this blog article with my opinion that code should be written by humans.
AI is, and will never be, a replacement for human software development.
And I have the same sentiment with [this (relatively) recent post by Andrej Karpathy](https://x.com/karpathy/status/2015883857489522876)
on how using AI to code atropies coding skills.
I also think being a good coder is necessary to code with AI in the first place.
Structuring code, auditing, reviewing still needs to be done by people.
And the only way to sustain those skills is coding without AI assistance.

That being said, I think AI assisted coding is a great tool, and it's here to stay.
(Might be the only thing in the current AI bubble that will stay for a long time.)
My ideal AI assisted workflow [aligns with Primeagen's 99](https://www.youtube.com/watch?v=ws9zR-UzwTE).
However, I will admit that I have skill issues currently in any language that isn't python.
This, along with being an AI Engineer at a company where the main policy is to only develop using AI
has me using ClaudeCode most of the time.
I want to document my workflow using claude for development in this blog post.
This will be a work in progress, as I improve and adapt new workflows.

## Usage philosophy

I currently hold the following beliefs when it comes to AI coding;

- Skills and MCPs are not actually very useful steering for AI right now.
[Here is the blog post by vercel](https://vercel.com/blog/agents-md-outperforms-skills-in-our-agent-evals)
- I don't think swarms or other orchestration tools are very useful.
It seems like throwing a lot of compute to cover up for not planning enough.
- AI must be reigned in as much as possible.
Always review all changed code.
- I'm weary of any new documented workflow that uses AI.
Most of this new workflow seems to be spearheaded by devs doing work that is already solved,
and offload cognitive load of architecture planning to agents.
- Whenever a new tool is available, I let the usage landscape settle before delving into it.

I think my workflow makes me much slower than other people when it comes to AI assisted coding,
but overall the workability with the codebase seems better when going slow.
At least to me.

## Currently on my try list

- Ralph framework
- Codex

## Dev Environment

### Flake

It's no secret that I use nix for all my dev environment management.
I have something analogous to the following dev shell in my `flake.nix`.
This ensures that I have both `uv` and `pnpm` in my environment.

```flake.nix
...
devShells.default = pkgs.mkShell {
  name = "<project>";
  
  buildInputs = with pkgs; [
    # Node.js (for compatibility with some npm packages and tooling)
    nodejs-slim
    pnpm
    # UV
    uv
    
    # Git for version control
    git
    
    # Development utilities
    curl
    wget
    
    # Shell utilities
    jq
    
    # Process management (useful for running dev servers)
    tmux
  ];

  shellHook = ''
    # Setup node
    export PATH="./node_modules/.bin:$PATH"
    
    # Disable telemetry for privacy (optional)
    export DO_NOT_TRACK=1
    export DISABLE_TELEMETRY=1
  '';
};
...
```

I have claude installed with pnpm; `pnpm -g update @anthropic-ai/claude-code`.
Providing claude (or other AI tools) through nix is a no-go for me since these tools get updated fairly often.

### Direnv

I have direnv setup in my system, to provide the working environment with project dependencies.
I usually default to the following `.envrc` for loading my dev shell.

```.envrc
# ENVRC: Set the dev environment

# Figure out our current worktree's root dir
export GIT_WT_DIR="$(git rev-parse --show-toplevel 2>/dev/null)"
if [ -n "$GIT_WT_DIR" ]; then
  # If our .git is a directory, then we are in the main worktree
  if [ -d "$GIT_WT_DIR/.git" ]; then
    export GIT_WT_MAIN_DIR="$GIT_WT_DIR"
  else
    # .git is a file, get the path to the common main worktree package
    gitdir_path="$(cat "$GIT_WT_DIR/.git" | sed -n 's|^gitdir: ||p')"
    # Main worktree is always the parent directory of .git
    export GIT_WT_MAIN_DIR="$(dirname "$(dirname "$(dirname "$gitdir_path")")")"
  fi
fi

# Make sure we are up to date, without shooting for conflicts
git pull --ff-only

# Link CLAUDE.md and AGENTS.md
if [ -f "${GIT_WT_DIR}/AGENTS.md" ]; then
  if [ ! -f "${GIT_WT_DIR}/CLAUDE.md" ]; then
    ln -s "${GIT_WT_DIR}/AGENTS.md" "${GIT_WT_DIR}/CLAUDE.md"
  fi
fi

# Use relevant dev shell
branch=$(git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "")
case "${branch}" in
    'main' | 'master')  use flake .#default ;;
    'cuda' )            use flake .#cuda ;;
    *)                  use flake ;;
esac

# Load environment variables
ENV_FILES=(.env .env.development .env.local .env.development.local)
for f in "${ENV_FILES[@]}" ; do
    dotenv_if_exists "$GIT_WT_MAIN_DIR/$f"
done
```

The controversial statement here is probably the `git pull --ff-only` command.
I work in between computers a lot, and this helps.
The main downside of using direnv is that Cursor's built in terminal does not execute shell rc commands.
This is fixed currently by defaulting to legacy terminal option in Cursor settings.

## Claude Hooks

The main issue I have with claude is tool usage, as it always prefers to use tools I don't want in my projects.
The best way to deal with this is the claude hooks; which allows users to block tool calls.
I used [glennmatlin/claude_code_hooks_for_uv.md](https://gist.github.com/glennmatlin/fadc41edc3bb9ff68ff9cfa5d6b8aca7)
gist as a starting point, and implemented hooks that force;

- All python commands to be run through uv
- All node package management to go through pnpm instead of npm

My current plan is to build a git repo management tool,
and automatically commit static files under `~/.claude` in VCS, and keep them synced across computers.
