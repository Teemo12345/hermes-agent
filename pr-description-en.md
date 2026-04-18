# PR Description: Official Documentation Chinese Translation and Technical Fixes

## Translation Content

This commit completes the comprehensive Chinese translation of Hermes Agent official documentation, covering the following core modules:

### User Guide
- **Configuration** (`configuration.md`): Full translation of all configuration options, including terminal backends, context compression, auxiliary models, etc.
- **Skills System** (`features/skills.md`): Translation of skill management, skills hub, security scanning, etc.
- **Session Management** (`sessions.md`): Translation of session recovery, naming, search functionality, etc.
- **Security** (`security.md`): Translation of dangerous command approval, user authorization, container isolation, etc.
- **Nix & NixOS Installation** (`getting-started/nix-setup.md`): Translation of deployment mode comparison and configuration guide
- **Messaging Platform Integration**: Translation of configuration documents for WhatsApp, Telegram, Discord, etc.

### Translation Approach
- AI-assisted translation to ensure terminology consistency and technical accuracy
- Preserved filenames and links in English to ensure system compatibility
- Maintained document structure unchanged to ensure one-to-one correspondence with English version
- Optimized technical term translation to ensure accurate and understandable Chinese expression

## Technical Fixes

### Dependency Version Fix
- Fixed `@docusaurus/theme-mermaid` version conflict, pinned from ^3.9.2 to 3.9.2
- Removed `@easyops-cn/docusaurus-search-local` plugin to resolve build errors

### Configuration Optimization
- Commented out search plugin configuration in `docusaurus.config.ts` to resolve ProgressPlugin parameter errors
- Ensured all dependency versions are consistent with `@docusaurus/core@3.9.2`

## Notes
- Translation files are located in the `website/i18n/zh-Hans/` directory
- All filenames and links remain in English to avoid path and reference errors
- Technical term translations reference industry standards to ensure consistency
- Document structure completely corresponds to the English version for easy maintenance and updates

## Commit History
- 26ddfc4d: Completed configuration document translation and dependency version fixes
- 39f79485: Completed skills system and session management document translation
- [Other commits]: Completed security documentation and messaging platform integration document translation
- [Other commits]: Optimized translation quality and technical term accuracy

This PR aims to provide complete Hermes Agent documentation for Chinese users, while fixing dependency issues during the build process to ensure the documentation site can be built and deployed normally.