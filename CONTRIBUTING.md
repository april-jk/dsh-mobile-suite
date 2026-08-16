# Contributing

DSH Mobile Suite is split into independently versioned repositories. Open code changes in the repository that owns the affected component:

- Mobile app: <https://github.com/april-jk/dsh-mobile>
- DSH plugin: <https://github.com/april-jk/dsh-mobile-plugin>
- Relay: <https://github.com/april-jk/dsh-relay>
- Cross-component contracts and coordinated releases: this repository

Clone with submodules:

```bash
git clone --recurse-submodules https://github.com/april-jk/dsh-mobile-suite.git
```

For behavior that changes a cross-component contract, update the versioned document under `dsh-公共文档/` before or with the component changes. Preserve tunnel envelope `v: 1` compatibility unless a migration plan is included.

Before opening a pull request, run the checks documented in the affected component README. Do not commit credentials, Android signing keys, `.env` files, generated SQLite databases, or Railway tokens.

This is an unofficial community project. Contributions must not imply DeepSeek review, certification, endorsement, or support.
