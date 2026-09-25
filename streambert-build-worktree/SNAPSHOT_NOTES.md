# streambert-build-worktree — snapshot notes

Raw copy of the local `/home/user/streambert-build` working checkout as it existed on 2026-09-25.
Known repair applied at archive time (documented, not hidden):
- `expo-app/modules/expo-external-player/expo-module.config.json` was rolled back
  to the buggy object-form by a sandbox snapshot restore (3rd occurrence of this
  environmental corruption). Written here in the CORRECT string-FQCN form,
  identical to `main` @ 7473996 (see streambert-app-source/ or branch `main`).
- `.git`, `node_modules/`, `android/` were wiped by snapshot evictions and are
 not present (regenerate: `npm ci`, root and expo-app; `npx expo prebuild`).
