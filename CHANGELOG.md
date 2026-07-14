# Changelog

## [1.12.1](https://github.com/bffless/skills/compare/skills-v1.12.0...skills-v1.12.1) (2026-07-14)


### Bug Fixes

* **auth skill:** correct custom-domain relay for localhost (targetOrigin) ([d420a9d](https://github.com/bffless/skills/commit/d420a9d5fb37ab569fc4b119bdc19442e605de91))
* **auth skill:** correct custom-domain relay for localhost (targetOrigin) ([edc14c8](https://github.com/bffless/skills/commit/edc14c8dc10a546add55d20ef8845aa20098c543))

## [1.12.0](https://github.com/bffless/skills/compare/skills-v1.11.0...skills-v1.12.0) (2026-07-14)


### Features

* **rules-as-code:** document rules init and name-based schema sync ([71769df](https://github.com/bffless/skills/commit/71769dfc3d3d49328bdb8304fd21a4e1d8c2639c))
* **rules-as-code:** document rules init and name-based schema sync ([3bf0497](https://github.com/bffless/skills/commit/3bf049776e13e38284765d5d5209df103075e73c))

## [1.11.0](https://github.com/bffless/skills/compare/skills-v1.10.2...skills-v1.11.0) (2026-07-12)


### Features

* rules-as-code skill (CLI, CI sync, PR previews, rollback) ([c261133](https://github.com/bffless/skills/commit/c261133dff1d9ef543e4fc685c52849041e2b5eb))
* rules-as-code skill (CLI, CI sync, PR previews, rollback) ([9e16206](https://github.com/bffless/skills/commit/9e162069839fb0c35fd8722d19405baa3b72cd8b))


### Bug Fixes

* correct rules dev/revisions/rollback flag coverage in rules-as-code skill ([2174752](https://github.com/bffless/skills/commit/21747521ce44d59bb052fcf2cfd8f574ef0e1e2e))

## [1.10.2](https://github.com/bffless/skills/compare/skills-v1.10.1...skills-v1.10.2) (2026-07-04)


### Documentation

* **pipelines:** document ai_handler systemPrompt vs messageField syntax ([1e4afb9](https://github.com/bffless/skills/commit/1e4afb9090cda89bd6f5aaf2d1f9d95c6f150468))
* **pipelines:** document ai_handler systemPrompt vs messageField syntax ([d6676bb](https://github.com/bffless/skills/commit/d6676bb1107caff3f8458bc1132d93d030171fcb))
* **pipelines:** warn against routing large result sets through function_handler ([baea37c](https://github.com/bffless/skills/commit/baea37c16f7ed22e96ff4720c323d674d6571a32))
* **pipelines:** warn against routing large result sets through function_handler ([caff3c6](https://github.com/bffless/skills/commit/caff3c628a86ea3554f76fcf8a1b9effedb49858))

## [1.10.1](https://github.com/bffless/skills/compare/skills-v1.10.0...skills-v1.10.1) (2026-06-28)


### Documentation

* **authentication:** API key → session for headless automation ([59b96d6](https://github.com/bffless/skills/commit/59b96d65976f24a9174d25f74559b98d055a58bd))
* **authentication:** document API key → session for headless automation ([556db68](https://github.com/bffless/skills/commit/556db683b6f331189113d83a2a6529a5f70f8a5e))

## [1.10.0](https://github.com/bffless/skills/compare/skills-v1.9.2...skills-v1.10.0) (2026-06-28)


### Features

* add pipeline-to-skill generator skill ([83febf2](https://github.com/bffless/skills/commit/83febf238f6bbe2ea0845cdc6a06cb101fdd3d30))
* pipeline-to-skill generator skill ([80f4444](https://github.com/bffless/skills/commit/80f4444cbaefbc4d4c6317ba0a2a4c38705436c0))


### Bug Fixes

* discover MCP/domain per environment instead of hardcoding j5s-dev ([a6bcebf](https://github.com/bffless/skills/commit/a6bcebf3a172020ceb620f9a0ed955d3f45caf32))
* generator output is a portable SKILL.md, not Claude-only ([3b1000d](https://github.com/bffless/skills/commit/3b1000de84c026691b006e7441eeda0440893277))
* make generated auth runtime-neutral (not Claude-only) ([204911a](https://github.com/bffless/skills/commit/204911ae5882c8d0e00b1e574ac14369a79846db))

## [1.9.2](https://github.com/bffless/skills/compare/skills-v1.9.1...skills-v1.9.2) (2026-06-27)


### Documentation

* **proxy-rules:** require multi-line function_handler code ([159027b](https://github.com/bffless/skills/commit/159027b0d9cd93f635755bf46186b63807fa1e82))
* **proxy-rules:** require multi-line function_handler code ([17724cb](https://github.com/bffless/skills/commit/17724cb4cea7f9c00f92718351318b5b1c10afcd))

## [1.9.1](https://github.com/bffless/skills/compare/skills-v1.9.0...skills-v1.9.1) (2026-06-18)


### Bug Fixes

* **skills:** keep visual-plans gallery cards from overflowing ([f8cbaa0](https://github.com/bffless/skills/commit/f8cbaa0e7ed7fb0593b06a8f3ddbaac90f56fa3d))
* **skills:** keep visual-plans gallery cards from overflowing ([b54317b](https://github.com/bffless/skills/commit/b54317b0862044157b231ac08f4d7b87cb927600))

## [1.9.0](https://github.com/bffless/skills/compare/skills-v1.8.4...skills-v1.9.0) (2026-06-18)


### Features

* **skills:** add visual-plans and visual-recaps skills ([1e74104](https://github.com/bffless/skills/commit/1e74104c535f16930ee30f29b1c6a73a31fce32c))
* **skills:** add visual-plans and visual-recaps skills ([c15dd50](https://github.com/bffless/skills/commit/c15dd50e239c4072979e38f044a70c7389bf4592))

## [1.8.4](https://github.com/bffless/skills/compare/skills-v1.8.3...skills-v1.8.4) (2026-06-05)


### Documentation

* **pipelines:** document presigned direct-to-bucket upload handlers ([c8c1381](https://github.com/bffless/skills/commit/c8c1381f49acf6e8e81dad9d7f411f4617043a1b))
* **pipelines:** document presigned direct-to-bucket upload handlers ([f32c272](https://github.com/bffless/skills/commit/f32c27268f1e6d13b3a504417f83141b1a4a8958))

## [1.8.3](https://github.com/bffless/skills/compare/skills-v1.8.2...skills-v1.8.3) (2026-06-04)


### Documentation

* **authentication:** don't use customDomainRelay for the primary domain ([7c99672](https://github.com/bffless/skills/commit/7c99672181bcab8d4c3f9637c92dd83a7c6ee412))
* **authentication:** don't use customDomainRelay for the primary domain ([e5e0581](https://github.com/bffless/skills/commit/e5e058189090033b9b1d1b25e7f790e4d2b13b42))
* **pipelines:** require multi-line indented handler code via MCP ([f73c05e](https://github.com/bffless/skills/commit/f73c05e8c9b4f2e36c8224e485a1c54ef9b83a90))

## [1.8.2](https://github.com/bffless/skills/compare/skills-v1.8.1...skills-v1.8.2) (2026-06-03)


### Documentation

* **authentication:** document reverse-proxy escape hatch + primary-domain framing ([f3b88ed](https://github.com/bffless/skills/commit/f3b88ede963167798125faef049d76272c8ab7f0))
* **authentication:** document reverse-proxy to SuperTokens for advanced cases ([187d128](https://github.com/bffless/skills/commit/187d12891b5076f007688ecfe712a415e7e3de16))

## [1.8.1](https://github.com/bffless/skills/compare/skills-v1.8.0...skills-v1.8.1) (2026-06-03)


### Documentation

* **authentication:** fix common bugs and add local-dev + logout patterns ([32b2efc](https://github.com/bffless/skills/commit/32b2efc995a24af093b509d174aa0eb335c91f67))
* **authentication:** fix common bugs and add local-dev + logout patterns ([7fdb466](https://github.com/bffless/skills/commit/7fdb4668946f686ed256c93c7d8d7f7d8e4be6a5))

## [1.8.0](https://github.com/bffless/skills/compare/skills-v1.7.0...skills-v1.8.0) (2026-06-01)


### Features

* document multiple proxy rule sets on upload-artifact ([506bfdf](https://github.com/bffless/skills/commit/506bfdffc0af65402dc8830a1865480e5c92822f))
* document multiple proxy rule sets on upload-artifact ([9a5c943](https://github.com/bffless/skills/commit/9a5c943c5030a00763bb366b5edfe94f8ba62778))


### Bug Fixes

* base path skills ([619b4e2](https://github.com/bffless/skills/commit/619b4e26d682c7dfa6bcabc635d2c25c0ce92be7))

## [1.7.0](https://github.com/bffless/skills/compare/skills-v1.6.0...skills-v1.7.0) (2026-05-26)


### Features

* add missing handler docs to pipelines skill and update proxy-rules ([400dfab](https://github.com/bffless/skills/commit/400dfab54440c5e58a15416864bdd23620447ee6))
* adds auth skill ([ddc1e68](https://github.com/bffless/skills/commit/ddc1e68db7bd89cef8c135453837022a0db4fc48))
* adds cache and use-bff-state skills ([22db53a](https://github.com/bffless/skills/commit/22db53ac595c1a0042df60b35e4d012e1d24ad02))
* adds group to aggregate ([7ab9a8c](https://github.com/bffless/skills/commit/7ab9a8cf390ee89ca6c292331951800abc237a75))
* initial commit ([b31c713](https://github.com/bffless/skills/commit/b31c7133ea4ceeed33b8ab484a0ac588ac57203e))
* renames claude-skills to skills ([bd4e8ee](https://github.com/bffless/skills/commit/bd4e8eefbdc87d80c1546ef78817881a3b7c9168))


### Bug Fixes

* ci ([b3affaf](https://github.com/bffless/skills/commit/b3affaf563cfd188745872c65bdc899400081042))
* exclude marketplace.json from npm package to prevent infinite recursion ([4d69692](https://github.com/bffless/skills/commit/4d6969293b9bee8054967505f747d4dabe337c2e))
* license ([f22bfe3](https://github.com/bffless/skills/commit/f22bfe3c52894ab56521d3eb2182d11371a19fd9))
* move plugin into subdirectory to match official plugin structure ([e5bbb59](https://github.com/bffless/skills/commit/e5bbb59119ceef3e290973fa24359a6c0e2f4f28))
* plugin ([17121b5](https://github.com/bffless/skills/commit/17121b52922b161cb7232cf514896d87725411fb))
* prevent plugin.json self-reference loop and use dynamic version ([c960725](https://github.com/bffless/skills/commit/c960725024e9576477eddf61e3ba69a3cc39a96f))
* recursive loop ([ebc5ef2](https://github.com/bffless/skills/commit/ebc5ef2344a07161be9a2dc8b040714f0992bcc9))
* remove self-referencing plugin.json from npm package ([f4b8e3a](https://github.com/bffless/skills/commit/f4b8e3a6153a1ee9390c737264030b9c66206d73))
* rename marketplace to bffless-plugins so plugin can be namespaced as bffless ([b34ec53](https://github.com/bffless/skills/commit/b34ec538543d2a2d443af5ddc6d7b67d0bb061e9))
* sync marketplace.json version with release-please ([09a0b57](https://github.com/bffless/skills/commit/09a0b57d69e6ede3eed966a851663465283c47a4))
* sync plugin.json version and add to release-please extra-files ([22bfbda](https://github.com/bffless/skills/commit/22bfbda745218a75ea6cd8a846b3a7abc835cfd3))
* tighten local-dev instructions wording in README ([b248fa8](https://github.com/bffless/skills/commit/b248fa8c89682251812d668708fe14a22323a47a))
* tighten local-dev instructions wording in README ([1ee271e](https://github.com/bffless/skills/commit/1ee271e09737a3a0609b2b635de10d9499cc642b))
* use relative path source instead of npm to prevent recursive caching ([ec8b071](https://github.com/bffless/skills/commit/ec8b071374eaf6df1a8c7beffddbf1462da33f18))


### Miscellaneous

* **main:** release claude-skills 1.1.0 ([005fc7a](https://github.com/bffless/skills/commit/005fc7a3454627d5e6c4ec1933a3aa0058fb8f6b))
* **main:** release claude-skills 1.1.0 ([31d2147](https://github.com/bffless/skills/commit/31d2147f7d2dd29e7643c6bc6a21b828edb8a9b3))
* **main:** release claude-skills 1.1.1 ([9d0f166](https://github.com/bffless/skills/commit/9d0f166295862f92e49db2530dbe240c3a84a5a6))
* **main:** release claude-skills 1.1.1 ([dbb3e48](https://github.com/bffless/skills/commit/dbb3e484ad764f67e4b793d810191d77c08ca30c))
* **main:** release claude-skills 1.1.2 ([4ae4d8c](https://github.com/bffless/skills/commit/4ae4d8cf8837b64c8df3cb9cf7bfb2c21d3b6e7b))
* **main:** release claude-skills 1.1.2 ([7f60591](https://github.com/bffless/skills/commit/7f60591428fac9dcdeb069fcffdb66c59a43d9a3))
* **main:** release claude-skills 1.2.0 ([f8dae4d](https://github.com/bffless/skills/commit/f8dae4db030a90fce1ac3044a916e06741f26058))
* **main:** release claude-skills 1.2.0 ([6e2f78c](https://github.com/bffless/skills/commit/6e2f78c0d757866918e273d33a66ae265be31734))
* **main:** release claude-skills 1.2.1 ([fc88d02](https://github.com/bffless/skills/commit/fc88d0264e7666c4b1cca0a80b999f534d93b5a2))
* **main:** release claude-skills 1.2.1 ([8ab70cf](https://github.com/bffless/skills/commit/8ab70cf952d3a3957263e7e75200ce6f64d074df))
* **main:** release claude-skills 1.2.2 ([ed4cdce](https://github.com/bffless/skills/commit/ed4cdce695fe478ba4ae0b6ca6a7aafc71d11d0b))
* **main:** release claude-skills 1.2.2 ([b4df1ad](https://github.com/bffless/skills/commit/b4df1ad44d78706569488b8598802e2862290e87))
* **main:** release claude-skills 1.2.3 ([8001f38](https://github.com/bffless/skills/commit/8001f38cc9830d1e371ef549635ee82a4d7a9b57))
* **main:** release claude-skills 1.2.3 ([becf818](https://github.com/bffless/skills/commit/becf8184f9c3cdac65ba100e7c98dba1910073ca))
* **main:** release claude-skills 1.3.0 ([bd129cd](https://github.com/bffless/skills/commit/bd129cde5874f10f8acee711a26184472a4ecf94))
* **main:** release claude-skills 1.3.0 ([ff0096d](https://github.com/bffless/skills/commit/ff0096d681963db11c82027c498d0b357518d161))
* **main:** release claude-skills 1.4.0 ([378df60](https://github.com/bffless/skills/commit/378df60d1c35586fb09ade76b7f82cbb1c3a5e8e))
* **main:** release claude-skills 1.4.0 ([de30794](https://github.com/bffless/skills/commit/de307947d071ef8e7e5abba84e3febfc6fdb55d3))
* **main:** release claude-skills 1.4.1 ([c9ce474](https://github.com/bffless/skills/commit/c9ce474ce73dadd28e2ea98333bb5ed718aabef5))
* **main:** release claude-skills 1.4.1 ([c174ea2](https://github.com/bffless/skills/commit/c174ea2b7b0f267cec328bd13ad9a988e10e8bdc))
* **main:** release claude-skills 1.4.2 ([7cd4f8b](https://github.com/bffless/skills/commit/7cd4f8be06769916d1c557423ca97bc3752ec181))
* **main:** release claude-skills 1.4.2 ([6317a6c](https://github.com/bffless/skills/commit/6317a6ca6a13c297178b7104d5e324acef4fb73f))
* **main:** release claude-skills 1.5.0 ([c28eca0](https://github.com/bffless/skills/commit/c28eca0b0816795b6f475f7398313d888c3b3f49))
* **main:** release claude-skills 1.5.0 ([7011276](https://github.com/bffless/skills/commit/7011276469f859c3886e7c8667c795ec38bf6546))
* **main:** release claude-skills 1.5.1 ([3f0a193](https://github.com/bffless/skills/commit/3f0a1937aab8833417cd477be9c7983e52bcf248))
* **main:** release claude-skills 1.5.1 ([1998594](https://github.com/bffless/skills/commit/19985945cb30b8c3a7d0efcb9246f69470a3a53b))
* **main:** release claude-skills 1.5.2 ([8b7deaa](https://github.com/bffless/skills/commit/8b7deaa59d49dde5f7c0c5d5b82c26093bf678a7))
* **main:** release claude-skills 1.5.2 ([d7cd874](https://github.com/bffless/skills/commit/d7cd874347c3e481f08af687f29bbb2660608f26))
* **main:** release claude-skills 1.5.3 ([5ab51a2](https://github.com/bffless/skills/commit/5ab51a29326d667502a616aeb36a287ed1e463f3))
* **main:** release claude-skills 1.5.3 ([d4d63b6](https://github.com/bffless/skills/commit/d4d63b6266ed24788209f573104bb40f31bbe733))
* **main:** release claude-skills 1.6.0 ([f4d1fa0](https://github.com/bffless/skills/commit/f4d1fa094771e7808e71ef1c5a0b139c89d87e1b))
* **main:** release claude-skills 1.6.0 ([5aa1485](https://github.com/bffless/skills/commit/5aa14854fc02f11df1052fb4abdeb5e237c809a7))
* workflow dispatch ([4775780](https://github.com/bffless/skills/commit/47757809a26943106a91c9fe21ce232b12897f56))

## [1.6.0](https://github.com/bffless/skills/compare/skills-v1.5.3...skills-v1.6.0) (2026-04-11)

### Features

- adds cache and use-bff-state skills ([22db53a](https://github.com/bffless/skills/commit/22db53ac595c1a0042df60b35e4d012e1d24ad02))

## [1.5.3](https://github.com/bffless/skills/compare/skills-v1.5.2...skills-v1.5.3) (2026-04-04)

### Bug Fixes

- sync plugin.json version and add to release-please extra-files ([22bfbda](https://github.com/bffless/skills/commit/22bfbda745218a75ea6cd8a846b3a7abc835cfd3))

## [1.5.2](https://github.com/bffless/skills/compare/skills-v1.5.1...skills-v1.5.2) (2026-04-04)

### Bug Fixes

- rename marketplace to bffless-plugins so plugin can be namespaced as bffless ([b34ec53](https://github.com/bffless/skills/commit/b34ec538543d2a2d443af5ddc6d7b67d0bb061e9))

## [1.5.1](https://github.com/bffless/skills/compare/skills-v1.5.0...skills-v1.5.1) (2026-04-04)

### Bug Fixes

- move plugin into subdirectory to match official plugin structure ([e5bbb59](https://github.com/bffless/skills/commit/e5bbb59119ceef3e290973fa24359a6c0e2f4f28))

## [1.5.0](https://github.com/bffless/skills/compare/skills-v1.4.2...skills-v1.5.0) (2026-04-04)

### Features

- add missing handler docs to pipelines skill and update proxy-rules ([400dfab](https://github.com/bffless/skills/commit/400dfab54440c5e58a15416864bdd23620447ee6))
- adds auth skill ([ddc1e68](https://github.com/bffless/skills/commit/ddc1e68db7bd89cef8c135453837022a0db4fc48))
- adds group to aggregate ([7ab9a8c](https://github.com/bffless/skills/commit/7ab9a8cf390ee89ca6c292331951800abc237a75))
- initial commit ([b31c713](https://github.com/bffless/skills/commit/b31c7133ea4ceeed33b8ab484a0ac588ac57203e))

### Bug Fixes

- exclude marketplace.json from npm package to prevent infinite recursion ([4d69692](https://github.com/bffless/skills/commit/4d6969293b9bee8054967505f747d4dabe337c2e))
- license ([f22bfe3](https://github.com/bffless/skills/commit/f22bfe3c52894ab56521d3eb2182d11371a19fd9))
- plugin ([17121b5](https://github.com/bffless/skills/commit/17121b52922b161cb7232cf514896d87725411fb))
- prevent plugin.json self-reference loop and use dynamic version ([c960725](https://github.com/bffless/skills/commit/c960725024e9576477eddf61e3ba69a3cc39a96f))
- recursive loop ([ebc5ef2](https://github.com/bffless/skills/commit/ebc5ef2344a07161be9a2dc8b040714f0992bcc9))
- remove self-referencing plugin.json from npm package ([f4b8e3a](https://github.com/bffless/skills/commit/f4b8e3a6153a1ee9390c737264030b9c66206d73))
- sync marketplace.json version with release-please ([09a0b57](https://github.com/bffless/skills/commit/09a0b57d69e6ede3eed966a851663465283c47a4))
- use relative path source instead of npm to prevent recursive caching ([ec8b071](https://github.com/bffless/skills/commit/ec8b071374eaf6df1a8c7beffddbf1462da33f18))

### Miscellaneous

- **main:** release skills 1.1.0 ([005fc7a](https://github.com/bffless/skills/commit/005fc7a3454627d5e6c4ec1933a3aa0058fb8f6b))
- **main:** release skills 1.1.0 ([31d2147](https://github.com/bffless/skills/commit/31d2147f7d2dd29e7643c6bc6a21b828edb8a9b3))
- **main:** release skills 1.1.1 ([9d0f166](https://github.com/bffless/skills/commit/9d0f166295862f92e49db2530dbe240c3a84a5a6))
- **main:** release skills 1.1.1 ([dbb3e48](https://github.com/bffless/skills/commit/dbb3e484ad764f67e4b793d810191d77c08ca30c))
- **main:** release skills 1.1.2 ([4ae4d8c](https://github.com/bffless/skills/commit/4ae4d8cf8837b64c8df3cb9cf7bfb2c21d3b6e7b))
- **main:** release skills 1.1.2 ([7f60591](https://github.com/bffless/skills/commit/7f60591428fac9dcdeb069fcffdb66c59a43d9a3))
- **main:** release skills 1.2.0 ([f8dae4d](https://github.com/bffless/skills/commit/f8dae4db030a90fce1ac3044a916e06741f26058))
- **main:** release skills 1.2.0 ([6e2f78c](https://github.com/bffless/skills/commit/6e2f78c0d757866918e273d33a66ae265be31734))
- **main:** release skills 1.2.1 ([fc88d02](https://github.com/bffless/skills/commit/fc88d0264e7666c4b1cca0a80b999f534d93b5a2))
- **main:** release skills 1.2.1 ([8ab70cf](https://github.com/bffless/skills/commit/8ab70cf952d3a3957263e7e75200ce6f64d074df))
- **main:** release skills 1.2.2 ([ed4cdce](https://github.com/bffless/skills/commit/ed4cdce695fe478ba4ae0b6ca6a7aafc71d11d0b))
- **main:** release skills 1.2.2 ([b4df1ad](https://github.com/bffless/skills/commit/b4df1ad44d78706569488b8598802e2862290e87))
- **main:** release skills 1.2.3 ([8001f38](https://github.com/bffless/skills/commit/8001f38cc9830d1e371ef549635ee82a4d7a9b57))
- **main:** release skills 1.2.3 ([becf818](https://github.com/bffless/skills/commit/becf8184f9c3cdac65ba100e7c98dba1910073ca))
- **main:** release skills 1.3.0 ([bd129cd](https://github.com/bffless/skills/commit/bd129cde5874f10f8acee711a26184472a4ecf94))
- **main:** release skills 1.3.0 ([ff0096d](https://github.com/bffless/skills/commit/ff0096d681963db11c82027c498d0b357518d161))
- **main:** release skills 1.4.0 ([378df60](https://github.com/bffless/skills/commit/378df60d1c35586fb09ade76b7f82cbb1c3a5e8e))
- **main:** release skills 1.4.0 ([de30794](https://github.com/bffless/skills/commit/de307947d071ef8e7e5abba84e3febfc6fdb55d3))
- **main:** release skills 1.4.1 ([c9ce474](https://github.com/bffless/skills/commit/c9ce474ce73dadd28e2ea98333bb5ed718aabef5))
- **main:** release skills 1.4.1 ([c174ea2](https://github.com/bffless/skills/commit/c174ea2b7b0f267cec328bd13ad9a988e10e8bdc))
- **main:** release skills 1.4.2 ([7cd4f8b](https://github.com/bffless/skills/commit/7cd4f8be06769916d1c557423ca97bc3752ec181))
- **main:** release skills 1.4.2 ([6317a6c](https://github.com/bffless/skills/commit/6317a6ca6a13c297178b7104d5e324acef4fb73f))

## [1.4.2](https://github.com/bffless/skills/compare/skills-v1.4.1...skills-v1.4.2) (2026-04-04)

### Bug Fixes

- use relative path source instead of npm to prevent recursive caching ([ec8b071](https://github.com/bffless/skills/commit/ec8b071374eaf6df1a8c7beffddbf1462da33f18))

## [1.4.1](https://github.com/bffless/skills/compare/skills-v1.4.0...skills-v1.4.1) (2026-04-04)

### Bug Fixes

- recursive loop ([ebc5ef2](https://github.com/bffless/skills/commit/ebc5ef2344a07161be9a2dc8b040714f0992bcc9))

## [1.4.0](https://github.com/bffless/skills/compare/skills-v1.3.0...skills-v1.4.0) (2026-04-04)

### Features

- adds group to aggregate ([7ab9a8c](https://github.com/bffless/skills/commit/7ab9a8cf390ee89ca6c292331951800abc237a75))

## [1.3.0](https://github.com/bffless/skills/compare/skills-v1.2.3...skills-v1.3.0) (2026-04-02)

### Features

- add missing handler docs to pipelines skill and update proxy-rules ([400dfab](https://github.com/bffless/skills/commit/400dfab54440c5e58a15416864bdd23620447ee6))

## [1.2.3](https://github.com/bffless/skills/compare/skills-v1.2.2...skills-v1.2.3) (2026-04-01)

### Bug Fixes

- prevent plugin.json self-reference loop and use dynamic version ([c960725](https://github.com/bffless/skills/commit/c960725024e9576477eddf61e3ba69a3cc39a96f))

## [1.2.2](https://github.com/bffless/skills/compare/skills-v1.2.1...skills-v1.2.2) (2026-04-01)

### Bug Fixes

- remove self-referencing plugin.json from npm package ([f4b8e3a](https://github.com/bffless/skills/commit/f4b8e3a6153a1ee9390c737264030b9c66206d73))

## [1.2.1](https://github.com/bffless/skills/compare/skills-v1.2.0...skills-v1.2.1) (2026-04-01)

### Bug Fixes

- sync marketplace.json version with release-please ([09a0b57](https://github.com/bffless/skills/commit/09a0b57d69e6ede3eed966a851663465283c47a4))

## [1.2.0](https://github.com/bffless/skills/compare/skills-v1.1.2...skills-v1.2.0) (2026-04-01)

### Features

- adds auth skill ([ddc1e68](https://github.com/bffless/skills/commit/ddc1e68db7bd89cef8c135453837022a0db4fc48))

### Bug Fixes

- license ([f22bfe3](https://github.com/bffless/skills/commit/f22bfe3c52894ab56521d3eb2182d11371a19fd9))

## [1.1.2](https://github.com/bffless/skills/compare/skills-v1.1.1...skills-v1.1.2) (2026-03-30)

### Bug Fixes

- plugin ([17121b5](https://github.com/bffless/skills/commit/17121b52922b161cb7232cf514896d87725411fb))

## [1.1.1](https://github.com/bffless/skills/compare/skills-v1.1.0...skills-v1.1.1) (2026-03-30)

### Bug Fixes

- exclude marketplace.json from npm package to prevent infinite recursion ([4d69692](https://github.com/bffless/skills/commit/4d6969293b9bee8054967505f747d4dabe337c2e))

## [1.1.0](https://github.com/bffless/skills/compare/skills-v1.0.0...skills-v1.1.0) (2026-03-30)

### Features

- initial commit ([b31c713](https://github.com/bffless/skills/commit/b31c7133ea4ceeed33b8ab484a0ac588ac57203e))
