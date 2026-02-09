
How to run the release GH Action
--------------------------------

Push a git tag (triggers on tags `v*`)

```bash
git tag -a v<version> -m "release v<version>"
git push origin v1.30
```