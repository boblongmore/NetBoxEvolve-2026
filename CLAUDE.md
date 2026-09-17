# Project Guidance

## Project Context
This project builds a common data model in NetBox, used across Arista AVD and Cisco NAC, for building multi-vendor Data Center Fabrics. It will be presented publicly at the NetBox Evolve Conference.

## Working Preferences
- Prefer simplicity over complexity. Avoid over-engineering, unnecessary abstractions, or clever solutions when a simple one works.
- Never commit or push anything to git without asking for permission first, even if a commit seems like a natural next step.
- Run linters often, and correct issues as they're found rather than letting them accumulate.

## Style Guide: The Zen of Ansible
Follow the [Zen of Ansible](https://www.redhat.com/en/blog/the-zen-of-ansible) for playbooks, roles, and automation code in this project:

1. Ansible is not Python.
2. YAML sucks for coding.
3. Playbooks are not for programming.
4. Ansible users are (most likely) not programmers.
5. Clear is better than cluttered.
6. Concise is better than verbose.
7. Simple is better than complex.
8. Readability counts.
9. Helping users get things done matters most.
10. User experience beats ideological purity.
11. "Magic" conquers the manual.
12. When giving users options, use convention over configuration.
13. Declarative is better than imperative -- most of the time.
14. Focus avoids complexity.
15. Complexity kills productivity.
16. If the implementation is hard to explain, it's a bad idea.
17. Every shell command and UI interaction is an opportunity to automate.
18. Just because something works, doesn't mean it can't be improved.
19. Friction should be eliminated whenever possible.
20. Automation is a journey that never ends.

## Documentation Style
- Documentation must be understandable by an audience of automation engineers and network engineers — not just software developers.
- This project will be presented publicly at the NetBox Evolve Conference, so documentation should be clear and presentable to that audience.
