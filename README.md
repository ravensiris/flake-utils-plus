
[![Discord](https://img.shields.io/discord/591914197219016707.svg?label=&logo=discord&logoColor=ffffff&color=7389D8&labelColor=6A7EC2)](https://discord.com/invite/RbvHtGa)

Need help? Create an issue or ping @Gytis#0001 in the above Discord Server.

# Changing branching policy #
From now on `master` serves as a development branch (previously `staging` was used for such purposes). Please use tags for stable releases of flake-utils-plus.
In general, with the improvements in test harness, releases might happen more frequently. Sticking with a tagged release might offer better trade-offs going forward.

Please note, while 1.2.0 retains backwards compatibility, 1.2.1 is the same version with all backwards compatibility removed.


# What is this flake? #

Flake-utils-plus exposes a library abstraction to *painlessly* generate NixOS flake configurations.

The biggest design goal is to keep down the fluff. The library is meant to be easy to understand and use. It aims to be far simpler than frameworks such as DevOS (previously called nixflk).

# Features of the flake #

Main flake-utils-plus features (Attributes visible from `flake.nix`):
- Extends [flake-utils](https://github.com/numtide/flake-utils). Everything exported by fu can be used from this flake.
- `lib.mkFlake { ... }` - Clean and pleasant to use flakes abstraction.
    - Option [`nix.generateRegistryFromInputs`](./lib/options.nix) - Generates `nix.registry` from flake inputs.
    - Option [`nix.generateNixPathFromInputs`](./lib/options.nix) - Generate `nix.nixPath` from available inputs.
    - Option [`nix.linkInputs`](./lib/options.nix) - Symlink inputs to /etc/nix/inputs.
    - Simple and clean support for multiple `nixpkgs` references.
    - `nixpkgs` references patching.
    - `channelsConfig` - Config applied to all `nixpkgs` references.
    - `hostDefaults` - Default configuration shared between host definitions.
    - `outputsBuilder` - Clean way to export packages/apps/etc.
    - `sharedOverlays` - Overlays applied on all imported channels.
- [`lib.exportModules [ ./a.nix ./b.nix ]`](./lib/exportModules.nix) - Generates module attribute which look like this `{ a = import ./a.nix; b = import ./b.nix; }`.
- [`lib.exportOverlays channels`](./lib/exportOverlays.nix) - Exports all overlays from channels as an appropriately namespaced attribute set. Users can instantiate with their nixpkgs version.
- [`lib.exportPackages self.overlays channels`](./lib/exportPackages.nix) - Similar to the overlay generator, but outputs them as packages for the platforms defined in `meta.platforms`. Unlike overlays, these packages are consistent across flakes allowing them to be cached.
- [`pkgs.fup-repl`](./lib/overlay.nix) - A package that adds a kick-ass repl. Usage:
    - `$ repl` - Loads your system repl into scope as well as `pkgs` and `lib` from `nixpkgs` input.
    - `$ repl /path/to/flake.nix` - Same as above but loads the specified flake.

# How to use #

* [Example of using multiple channels](./examples/minimal-multichannel)

* [Exporters usage example](./examples/exporters)

* [Using FUP to configure hosts with Home Manager, NUR and neovim](./examples/home-manager+nur+neovim)

## Examples

We recommend referring to people's examples below when setting up your system.

- [Gytis Dotfiles (Author of this project)](https://github.com/gytis-ivaskevicius/nixfiles/blob/master/flake.nix)
- [Fufexan Dotfiles](https://github.com/fufexan/dotfiles/blob/main/flake.nix)
- [Bobbbay Dotfiles](https://github.com/Bobbbay/dotfiles/blob/master/flake.nix)
- [Charlotte Dotfiles](https://github.com/chvp/nixos-config/blob/master/flake.nix)


# Documentation

Options with their example usage and description.

```nix
let
  inherit (builtins) removeAttrs;
  mkApp = utils.lib.mkApp;
  # If there is a need to get direct reference to nixpkgs - do this:
  pkgs = self.pkgs.x86_64-linux.nixpkgs;
in flake-utils-plus.lib.mkFlake {


  # `self` and `inputs` arguments are REQUIRED!
  inherit self inputs;

  # Supported systems, used for packages, apps, devShell and multiple other definitions. Defaults to `flake-utils.lib.defaultSystems`.
  supportedSystems = [ "x86_64-linux" ];


  ################
  ### channels ###
  ################

  # Configuration that is shared between all channels.
  channelsConfig = { allowBroken = true; };

  # Overlays which are applied to all channels.
  sharedOverlays = [ nur.overlay ];

  # Nixpkgs flake reference to be used in the configuration.
  # Autogenerated from `inputs` by default.
  channels.<name>.input = nixpkgs;

  # Channel specific config options.
  channels.<name>.config = { allowUnfree = true; };

  # Patches to apply on selected channel.
  channels.<name>.patches = [ ./someAwesomePatch.patch ];

  # Overlays to apply on a selected channel.
  channels.<name>.overlaysBuilder = channels: [
    (final: prev: { inherit (channels.unstable) neovim; })
  ];


  ####################
  ### hostDefaults ###
  ####################

  # Default architecture to be used for `hosts` defaults to "x86_64-linux".
  hostDefaults.system = "x86_64-linux";

  # Default modules to be passed to all hosts.
  hostDefaults.modules = [ ./module.nix ./module2 ];

  # Reference to `channels.<name>.*`, defines default channel to be used by hosts. Defaults to "nixpkgs".
  hostDefaults.channelName = "unstable";

  # Extra arguments to be passed to all modules. Merged with host's extraArgs.
  hostDefaults.extraArgs = { inherit utils inputs; foo = "foo"; };


  #############
  ### hosts ###
  #############

  # System architecture. Defaults to `defaultSystem` argument.
  hosts.<hostname>.system = "aarch64-linux";

  # <name> of the channel to be used. Defaults to `nixpkgs`;
  hosts.<hostname>.channelName = "unstable";

  # Extra arguments to be passed to the modules.
  hosts.<hostname>.extraArgs = { abc = 123; };

  # These are not part of the module system, so they can be used in `imports` lines without infinite recursion.
  hosts.<hostname>.specialArgs = { thing = "abc"; };

  # Host specific configuration.
  hosts.<hostname>.modules = [ ./configuration.nix ];

  # Flake output for configuration to be passed to. Defaults to `nixosConfigurations`.
  hosts.<hostname>.output = "darwinConfigurations";

  # System builder. Defaults to `channels.<name>.input.lib.nixosSystem`.
  # `removeAttrs` workaround due to this issue https://github.com/LnL7/nix-darwin/issues/319
  hosts.<hostname>.builder = args: nix-darwin.lib.darwinSystem (removeAttrs args [ "system" ]);


  #############################
  ### flake outputs builder ###
  #############################


  outputsBuilder = channels: {
    # Evaluates to `apps.<system>.custom-neovim  = utils.lib.mkApp { drv = ...; exePath = ...; };`.
    apps = {
      custom-neovim = mkApp {
        drv = fancy-neovim;
        exePath = "/bin/nvim";
      };
    };

    # Evaluates to `packages.<system>.coreutils = <unstable-nixpkgs-reference>.package-from-overlays`.
    packages = { inherit (channels.unstable) package-from-overlays; };

    # Evaluates to `apps.<system>.firefox  = utils.lib.mkApp { drv = ...; };`.
    defaultApp = mkApp { drv = channels.nixpkgs.firefox };

    # Evaluates to `defaultPackage.<system>.neovim = <nixpkgs-channel-reference>.neovim`.
    defaultPackage = channels.nixpkgs.neovim;

    # Evaluates to `devShell.<system> = <nixpkgs-channel-reference>.mkShell { name = "devShell"; };`.
    devShell = channels.nixpkgs.mkShell { name = "devShell"; };
  };


  #########################################################
  ### All other properties are passed down to the flake ###
  #########################################################

  checks.x86_64-linux.someCheck = pkgs.hello;
  packages.x86_64-linux.somePackage = pkss.hello;
  overlay = import ./overlays;
  abc = 132;

}
```