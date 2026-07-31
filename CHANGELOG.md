# Changelog

## [1.3.0](https://github.com/fortify/skills/compare/v1.2.0...v1.3.0) (2026-07-31)


### Features

* Add `fortify-exploitability-analysis` agent ([52fd54e](https://github.com/fortify/skills/commit/52fd54ed0adf85beeb796720eae9f26a2aa7aa01))


### Bug Fixes

* `fortify-exploitability-analysis` now references `fortify-dependency-upgrade` ([52fd54e](https://github.com/fortify/skills/commit/52fd54ed0adf85beeb796720eae9f26a2aa7aa01))
* Bump fcli version to 3.23 ([52fd54e](https://github.com/fortify/skills/commit/52fd54ed0adf85beeb796720eae9f26a2aa7aa01))
* Improve guardrails within skills ([52fd54e](https://github.com/fortify/skills/commit/52fd54ed0adf85beeb796720eae9f26a2aa7aa01))

## [1.2.0](https://github.com/fortify/skills/compare/v1.1.1...v1.2.0) (2026-06-29)


### Features

* add fortify-change-review and fortify-dependency-upgrade skills ([316b092](https://github.com/fortify/skills/commit/316b092cfaea0dd275721b5febe8ec3453fc7db0))
* disambiguate full scan/issue triage from change-review in fortify-fod ([a03c42b](https://github.com/fortify/skills/commit/a03c42bdc25b80e77ea42ae5c4a37ab803e57d8d))
* disambiguate full scan/issue triage from change-review in fortify-ssc ([75322ee](https://github.com/fortify/skills/commit/75322eec73c1b698dcd8a571775a27ce746f2e58))
* scope fortify-remediate to SAST/DAST, delegate SCA to fortify-dependency-upgrade ([bc6f3e2](https://github.com/fortify/skills/commit/bc6f3e2941a6c9bc5af86fcb8f6233584c821f1f))


### Bug Fixes

* bump fcli-common and fortify-create-app versions for sc-client fix ([b94c3b2](https://github.com/fortify/skills/commit/b94c3b24b607a9587b522960a8e08b15720067e5))
* Fix ScanCentral Client version handling ([7ad5fe2](https://github.com/fortify/skills/commit/7ad5fe24a6112865cf023ea212829522070b8b50))

## [1.1.1](https://github.com/fortify/skills/compare/v1.1.0...v1.1.1) (2026-05-21)


### Bug Fixes

* Fix version number handling ([ba70e9a](https://github.com/fortify/skills/commit/ba70e9a5dfa48b292f0c1e1c69f63261f46fa626))

## [1.1.0](https://github.com/fortify/skills/compare/v1.0.0...v1.1.0) (2026-05-21)


### Features

* Add fortify-create-app skill for guided app onboarding in FoD/SSC ([315d393](https://github.com/fortify/skills/commit/315d393f84fb8e33509839c73788cde4446894ab))
* Add fortify-exploitability-analysis skill for SCA CVE impact and reachability analysis ([315d393](https://github.com/fortify/skills/commit/315d393f84fb8e33509839c73788cde4446894ab))
* Add fortify-onboard agent for end-to-end application onboarding ([315d393](https://github.com/fortify/skills/commit/315d393f84fb8e33509839c73788cde4446894ab))
* **fortify-cicd-integration:** split platform references into FoD/SSC-specific variants ([315d393](https://github.com/fortify/skills/commit/315d393f84fb8e33509839c73788cde4446894ab))
* **fortify-fod:** extract mutating-operations and output-formats to shared reference files ([315d393](https://github.com/fortify/skills/commit/315d393f84fb8e33509839c73788cde4446894ab))
* **fortify-fod:** split app-or-release use case into dedicated create-release use case ([315d393](https://github.com/fortify/skills/commit/315d393f84fb8e33509839c73788cde4446894ab))
* **fortify-remediate:** add shared reference files (fcli-install, fcli-query-output, resolving-release, resolving-appversion) ([315d393](https://github.com/fortify/skills/commit/315d393f84fb8e33509839c73788cde4446894ab))
* **fortify-ssc:** extract mutating-operations and output-formats to shared reference files ([315d393](https://github.com/fortify/skills/commit/315d393f84fb8e33509839c73788cde4446894ab))
* **fortify-ssc:** split app-or-version use case into dedicated create-version use case ([315d393](https://github.com/fortify/skills/commit/315d393f84fb8e33509839c73788cde4446894ab))


### Bug Fixes

* **fcli-common:** streamline SKILL.md, extract session management to reference file ([315d393](https://github.com/fortify/skills/commit/315d393f84fb8e33509839c73788cde4446894ab))
* **fortify-fod:** simplify session verification and remove platform disambiguation ([315d393](https://github.com/fortify/skills/commit/315d393f84fb8e33509839c73788cde4446894ab))
* **fortify-ssc:** simplify session verification and remove platform disambiguation ([315d393](https://github.com/fortify/skills/commit/315d393f84fb8e33509839c73788cde4446894ab))
* register new skills and agent in plugin.json and update README ([315d393](https://github.com/fortify/skills/commit/315d393f84fb8e33509839c73788cde4446894ab))
* update shared reference file sync table in copilot-instructions.md ([315d393](https://github.com/fortify/skills/commit/315d393f84fb8e33509839c73788cde4446894ab))
