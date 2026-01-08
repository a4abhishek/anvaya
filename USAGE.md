
# Anvaya Usage

**Anvaya** is a centralized repository for learning documents, designed to be used standalone or as a submodule in any project.

## Using Anvaya as a Submodule

To use Anvaya in your project:

1. **Add as a submodule:**

  ```sh
  git submodule add git@github.com:a4abhishek/anvaya.git <documentation directory>/anvaya
  git submodule update --init --recursive
  ```

2. **Commit:** Commit the submodule addition to your main repository.
2. **Sync from remote:**

  ```sh
  git submodule update --remote --merge
  ```

  (Fetch latest changes from Anvaya)
4. **Update to remote:**

- Make changes in Anvaya (add/edit docs)
- Commit and push to the Anvaya repository

5. **Reference docs:** Link to Anvaya docs from your main project as needed

## General Usage

You can use Anvaya:

- **Standalone:**
  - Clone Anvaya directly and manage learning docs in a single repository.
  - Commit and push changes to the Anvaya repository.

- **As a Submodule:**
  - Add Anvaya to your project as described above.
  - Sync/pull updates from remote as needed.
  - Make changes, commit, and push to Anvaya.
  - Reference Anvaya docs from your main project.

## Best Practices

- Keep learning docs in Anvaya for all related projects.
- Reference Anvaya docs from your main project documentation as needed.
- Use submodule commands to keep Anvaya up-to-date.

## Troubleshooting

- If submodule is missing, run:

  ```sh
  git submodule update --init --recursive
  ```

- If you have merge conflicts, resolve them in the Anvaya repo and push.

---
For more details, see [Git Submodule Documentation](https://git-scm.com/book/en/v2/Git-Tools-Submodules).
