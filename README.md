# test-software-factory

Temporary, personal-account stand-in for the generic "software factory" repo — the SanioAI org repo is being created separately and isn't available to this account yet. Once that exists, this repo's contents move there and this one goes away.

**What lives here:** the reusable factory tooling — agent role definitions, artifact templates/schemas, and (eventually) the CLI, GitHub Actions workflows, and everything else that isn't specific to any one client project.

**What does NOT live here:** anything specific to Claris Commerce, Medusa, or any other target project. Those stay in their own product repos. This repo operates *on* target repos; it isn't one.

Test runs against a real target project use a disposable Spring Boot project (`usermgmt`), not `claris-commerce`, to avoid creating unnecessary commits/PRs in the real product repo while the factory process itself is still being validated.
