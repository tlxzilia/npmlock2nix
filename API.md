## API Documentation

The sections below describe the public API of _npmlock2nix_.

## Functions

### node_modules

The `node_modules` function parses `package.json` and `package-lock.json` and generates a derivation containing a populated `node_modules` folder equivalent to the result of running `npm install`.
#### Arguments
The `node_modules` function takes an attribute set with the following attributes:

- **src** *(mandatory)*: Path to the source containing `package.json` and `package-lock.json`
- **packageJson** *(default `src+"/package.json"`)*: Path to `package.json`
- **packageLockJson** *(default `src+"/package-lock.json"`)*: Path to `package-lock.json`
- **nodejs** *(default `nixpkgs.nodejs`, which is the Active LTS version)*: Node.js derivation to use
- **preInstallLinks** *(default `{}`)*: Map of symlinks to create inside npm dependencies in the `node_modules` output (See [Concepts](#concepts) for details).
- **githubSourceHashMap** *(default `{}`)*: Dependency hashes for evaluation in restricted mode (See [Concepts](#concepts) for details).
- **sourceOverrides** *(default `{}`)*: Derivation attributes to apply to sources, allowing patching (See the [source derivation overrides](#source-derivation-overrides) concept for details)

#### Notes
- You may provide additional arguments accepted by `mkDerivation` all of which are going to be passed on.
- Sometimes the installation behavior of npm packages needs to be altered by setting environment variables. You may set environment variables by simply adding them to the attribute set: `FOO_HOME = ${pkgs.libfoo.dev}`.


---

### shell
The `shell` function creates a nix-shell environment with the `node_modules` folder for the given npm project provided in the local directory as either copy or symlink (as determined by `node_modules_mode`).

#### Arguments
The `shell` function takes an attribute set with the following attributes:

- **src** *(mandatory)*: Path to the source containing `package.json` and `package-lock.json`
- **node_modules_mode** *(default `"symlink"`)*: Determines how the `node_modules` should be provided (See [Concepts](#concepts) for details).
- **node_modules_attrs** *(default `{}`)*: Overrides that will be passed to the `node_modules` function (See [Concepts](#concepts) for details).


#### Notes
- You may provide additional arguments accepted by `mkDerivation` all of which are going to be passed on.

---

### build
The `build` function creates a derivation for an arbitrary npm package by letting the user specify how to build and install it.

#### Arguments
The `build` function takes an attribute set with the following attributes:

- **src** *(mandatory)*: Path to the source containing `package.json` and `package-lock.json`.
- **installPhase** *(mandatory)*: Commands to install the package
- **buildCommands** *(default `["npm run build"]`)*: List of commands to build the package.
- **node_modules_attrs** *(default `{}`)*: Overrides that will be passed to the `node_modules` function (See [Concepts](#concepts) for details).
- **node_modules_mode** *(default `"symlink"`)*: Determines how the `node_modules` should be provided (See [Concepts](#concepts) for details).

#### Notes
- You may provide additional arguments accepted by `mkDerivation` all of which are going to be passed on.

## Concepts

### githubSourceHashMap
When _npmlock2nix_ is used in restricted evaluation mode (hydra for example), `node_modules` needs to be provided with the revision and sha256 of all GitHub dependencies via `githubSourceHashMap`:

```nix
npmlock2nix.node_modules {
  src = ./.;
  githubSourceHashMap = {
    tmcw.leftpad.db1442a0556c2b133627ffebf455a78a1ced64b9 = "1zyy1nxbby4wcl30rc8fsis1c3f7nafavnwd3qi4bg0x00gxjdnh";
  };
}
```

Please refer to [github-dependency](https://github.com/tweag/npmlock2nix/blob/master/tests/examples-projects/github-dependency/default.nix) for a fully working example.

### preInstallLinks

`preInstallLinks` was removed. Use source derivation overrides `sourceOverrides`


### node_modules_mode

_npmlock2nix_ can provide the `node_modules` folder to builds and development environment environments in two different ways as designated by `node_modules_mode`:

- `copy`: The `node_modules/` folder is copied from the nix store
- `symlink` The `node_modules/` folder is symlinked from the nix store

The first can be useful if you are also using `npm` to interactively update or modify your `node_modules` alongside nix based builds.

**Note**: If you are entering a shell environment using `npmlock2nix.shell` and there is an existing `node_modules/` _directory_ (instead of a symlink), `npmlock2nix` will print a warning but will not touch this directory. Symlinks on the other hand will be updated.

### node_modules_attrs

When you actually want to describe a shell environment or a build, but you need to pass attributes to `node_modules` you can do so by passing them via `node_modules_attrs` in both `build` and `shell`:

```nix
npmlock2nix.build {
  src = ./.;
  node_modules_attrs = {
    buildInputs = [ pkgs.zlib ];
  };
}
```

### Source derivation overrides

`node_modules` takes a `sourceOverrides` argument, which allows you to modify the source derivations of individual npm packages you depend on, mainly useful for adding Nix-specific fixes to packages. This could be used for patching interpreter or paths, or to replace vendored binaries with ones provided by Nix.

Npm packages binaries interpreter are automatically patched with environment shebangs when an entry exist for the npm package.

To replace `preInstallLinks` functionality. Symlinked vendored binaries are compiled to be added on node modules installation using npm preinstall script, this is required since npm strip symlinks from package archive.

The `sourceOverrides` argument expects an attribute set mapping npm package names to a function describing the modifications of that package. Each function receives an attribute set as a first argument, containing either a `version` attribute if the version is known, or a `github = { org, repo, rev, ref }` attribute if the package is fetched from GitHub. These values can be used to have different overrides depending on the version. The function receives another argument which is the derivation of the fetched source, which can be modified using `.overrideAttrs`. The fetched source mainly runs the [patch phase](https://nixos.org/manual/nixpkgs/stable/#ssec-patch-phase), so of particular interest are the `patches` and `postPatch` attributes, in which `patchShebangs` can be called.

`sourceOverrides` comes with default override mechanism like patching all npm packages binaries dependencies with `buildRequirePatchShebangs` or patching individual npm package binaries with `packageRequirePatchShebangs`.

```nix
npmlock2nix.node_modules {
  sourceOverrides = with npmlock2nix.node_modules; {
    # sourceInfo either contains:
    # - A version attribute
    # - A github = { org, repo, rev, ref } attribute for GitHub sources
    package-name = sourceInfo: drv: drv.overrideAttrs (old: {
      buildInputs = [ somePackage ];
      patches = [ somePatch ];
      postPatch = ''
        mkdir -p vendor
        ln -sf ${vendored_binary} ./vendor/vendored_binary
      '';
      # ...
    })

    # Patch all npm packages
    inherit buildRequirePatchShebangs

    # Patch individual package
    node-pre-gyp = packageRequirePatchShebangs

    # Link vendored binaries
    package-name = sourceInfo: drv: drv.overrideAttrs (old: {
      postPatch = ''
        mkdir -p vendor
        ln -sf ${vendored_binary} ./vendor/vendored_binary
      '';
    })

  };
}
```
