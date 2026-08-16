# STEPSS Scoop bucket

A [Scoop](https://scoop.sh/) bucket for **STEPSS**, *Static and Transient Electric
Power Systems Simulation*. It installs the desktop application on Windows and
keeps it current, so a new release arrives through `scoop update` instead of
being downloaded once per version.

```powershell
scoop bucket add sps-l https://github.com/SPS-L/stepss-scoop
scoop install stepss
```

Thereafter:

```powershell
scoop update stepss
```

Scoop installs into your own profile and needs **no administrator rights**,
which is the reason this route exists alongside the `.msi`: on a managed or
locked-down machine where an installer cannot be run, this one still works.

After installing you get STEPSS on the Start Menu under **Scoop Apps**, an icon
on the **desktop**, and a `stepss` command on `PATH`.

Uninstall with `scoop uninstall stepss`, which removes both shortcuts with it.

## Licensing

**STEPSS is not Apache 2.0 taken as a whole**, and the `license` field in the
manifest says so deliberately rather than copying the SPDX identifier off the
interface's own repository.

- **RAMSES**, the dynamic simulation engine, is the property of the University
  of Liège: proprietary, free for non-commercial use, capped at 1000 buses and
  2 cores.
- **Helios** and **CODEGEN** are under Academic Public Licences.
- The two user interfaces, **stepss-java-ui** and **stepss-python-ui**, are
  Apache 2.0.

<https://stepss.sps-lab.org/getting-started/license/> is the single owner of
these facts; everything here is a summary that points at it. Installing through
Scoop agrees to nothing, because the licence appears the first time you launch
STEPSS. Please do not mirror the bucket's release assets: the licence granted
for RAMSES is non-transferable.

## The Fortran toolchain is a separate, optional step

Compiling **your own** models with CODEGEN needs a Fortran toolchain. Nothing
else in STEPSS does, most users never will, and it is a large download, so the
manifest carries a `suggest` rather than a `depends` and installs nothing:

```powershell
scoop install msys2
msys2                 # then, in the MSYS2 shell that opens:
pacman -S mingw-w64-x86_64-gcc-fortran mingw-w64-x86_64-openblas make
```

STEPSS looks for a Scoop-installed MSYS2 at `%SCOOP%\apps\msys2\current` and
`%USERPROFILE%\scoop\apps\msys2\current` as well as at `C:\msys64`, so there is
no `MSYS2_ROOT` to set by hand. It searches every one of those that exists, not
just the first, because `scoop install msys2` does not displace an MSYS2 already
at `C:\msys64` and only one of the two is likely to be the one `pacman` ran in.

Three reasons this is not automated, beyond it being optional. Scoop's `depends`
can install MSYS2 but cannot drive `pacman` at all, so the half that actually
provides gfortran would have to be a `post_install` block whose first run wants
`pacman -Syu` and a shell restart, and whose failure leaves several hundred MB
half-installed with no rollback Scoop understands. And `.mod` files are readable
only by the gfortran release that wrote them, so installing whatever MSYS2's
rolling repository holds today would not even guarantee a *usable* toolchain,
only a present one.

## How the manifest stays current

`bucket/stepss.json` is rewritten by `.github/workflows/update-manifest.yml`,
which is triggered three ways:

| Trigger | When |
|---|---|
| `repository_dispatch` (`stepss-release`) | stepss-java-ui fires it from the `msi` leg of its release workflow, right after the portable zip is attached |
| `schedule`, weekly | insurance, because a dropped or 401ing dispatch is silent on this side |
| `workflow_dispatch` | by hand |

It rewrites exactly three fields, `version` plus `url` and `hash` under
`architecture.64bit`, and commits only if they moved. Everything else in the
manifest is hand-edited. The hash comes from the GitHub API's own per-asset
digest, so the workflow does not download 230 MB to compute a number GitHub
already has.

`checkver` and `autoupdate` are also present and correct, so
`bin\checkver.ps1 stepss -u` works if the automation is ever unavailable.

### Two things that will break this quietly

- **The release asset name is a contract.** The manifest resolves
  `STEPSS-<version>-windows.zip` on a stepss-java-ui release. Renaming it there
  breaks every release cut afterwards, and older tags stay un-resyncable because
  the new name never existed on them. Rename it in both repositories in one pass
  or not at all.
- **A published asset must stay where it is.** Scoop clients download an exact
  URL and verify an exact digest, so deleting or re-tagging a release someone
  may have installed breaks their install and their `scoop update` both.

### The layout the manifest depends on

The zip holds a single top-level `STEPSS/` directory, which `extract_dir`
flattens, and inside it the manifest needs two files by name:

- `STEPSS.exe`, the launcher, which `bin`, `shortcuts` and both hook scripts point at.
- `STEPSS.ico`, which is copied in beside it by stepss-java-ui's release
  workflow. Scoop **refuses to create a shortcut whose icon file is missing**,
  so losing it would fail every install rather than degrading to a plain icon.
  The `validate` job checks the manifest still names it, and the `update` job
  checks the zip still contains it before committing a version bump.

## Related

- [stepss-java-ui](https://github.com/SPS-L/stepss-java-ui) builds the artifact this serves
- [apt.sps-lab.org](https://apt.sps-lab.org/) is the same idea for Debian and Ubuntu
- [stepss.sps-lab.org](https://stepss.sps-lab.org/) is the documentation
