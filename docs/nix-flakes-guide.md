# Guide to Nix Flakes Patterns and Best Practices

## Introduction

This guide covers patterns and best practices for using Nix flakes in a pure way, without relying on external tools like flake-parts or flake-utils. We'll focus on modular design, reproducibility, and maintainability.

## Core Concepts

### 1. Flake Structure

Basic flake structure with modular organization:

```nix
{
  description = "A well-structured Nix flake";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    # Additional inputs as needed
  };

  outputs = { self, nixpkgs }: let
    # System-specific configuration
    supportedSystems = [ "x86_64-linux" "aarch64-linux" "x86_64-darwin" "aarch64-darwin" ];
    forAllSystems = nixpkgs.lib.genAttrs supportedSystems;
    
    # Import helper functions
    lib = import ./lib { inherit nixpkgs; };
    
    # Import modules
    modules = import ./modules;
    
    # Import overlays
    overlays = import ./overlays;
    
  in {
    # Your outputs here
  };
}
```

### 2. Module Organization

Structure your modules for reusability:

```
.
├── flake.nix
├── lib
│   ├── default.nix
│   ├── mkSystem.nix
│   └── utils.nix
├── modules
│   ├── default.nix
│   ├── core
│   │   └── default.nix
│   └── services
│       └── default.nix
├── overlays
│   ├── default.nix
│   └── packages.nix
└── packages
    └── default.nix
```

## Implementation Patterns

### 1. System Configuration

Create a reusable system configuration builder:

```nix
# lib/mkSystem.nix
{ nixpkgs }:
{ system
, modules ? []
, overlays ? []
}:

let
  pkgs = import nixpkgs {
    inherit system overlays;
    config.allowUnfree = true;
  };
in
nixpkgs.lib.nixosSystem {
  inherit system;
  modules = [
    # Core modules
    ./modules/core
  ] ++ modules;
  specialArgs = {
    inherit pkgs;
  };
}
```

### 2. Package Definitions

Define packages in a modular way:

```nix
# packages/default.nix
{ pkgs }:

let
  callPackage = pkgs.lib.callPackageWith (pkgs // self);
  self = {
    package1 = callPackage ./package1 {};
    package2 = callPackage ./package2 {};
  };
in self
```

### 3. Overlay Management

Organize overlays efficiently:

```nix
# overlays/default.nix
let
  # Import all overlay files
  overlayFiles = {
    packages = import ./packages.nix;
    custom = import ./custom.nix;
  };

  # Combine overlays
  combinedOverlays = final: prev:
    nixpkgs.lib.fold
      (overlay: acc: acc // (overlay final prev))
      {}
      (builtins.attrValues overlayFiles);
in
  combinedOverlays
```

### 4. Module Composition

Create composable NixOS modules:

```nix
# modules/services/myservice/default.nix
{ config, lib, pkgs, ... }:

with lib;

let
  cfg = config.services.myservice;
in {
  options.services.myservice = {
    enable = mkEnableOption "my custom service";
    
    port = mkOption {
      type = types.port;
      default = 8080;
      description = "Port to listen on";
    };
  };

  config = mkIf cfg.enable {
    systemd.services.myservice = {
      description = "My Custom Service";
      wantedBy = [ "multi-user.target" ];
      
      serviceConfig = {
        ExecStart = "${pkgs.myservice}/bin/myservice --port ${toString cfg.port}";
        Restart = "always";
      };
    };
  };
}
```

## Best Practices

### 1. Input Management

Handle inputs carefully:

```nix
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    
    # Pin specific versions for stability
    stable.url = "github:NixOS/nixpkgs/nixos-23.11";
    
    # Use flake-follows for consistent versioning
    myDep.url = "github:user/dep";
    myDep.inputs.nixpkgs.follows = "nixpkgs";
  };
}
```

### 2. Version Handling

Create a consistent version management system:

```nix
# lib/version.nix
let
  version = {
    major = 1;
    minor = 0;
    patch = 0;
    extra = "";
  };
in {
  inherit version;
  
  # Generate semantic version string
  versionStr = with version;
    "${toString major}.${toString minor}.${toString patch}${extra}";
    
  # Generate Nix-friendly version
  nixVersion = with version;
    "${toString major}${toString minor}${toString patch}";
}
```

### 3. Development Shell

Create reproducible development environments:

```nix
{
  outputs = { self, nixpkgs }: {
    devShells = forAllSystems (system:
      let
        pkgs = nixpkgs.legacyPackages.${system};
      in {
        default = pkgs.mkShell {
          buildInputs = with pkgs; [
            # Development tools
            rustc
            cargo
            rust-analyzer
            
            # Project-specific dependencies
            openssl
            pkg-config
          ];
          
          # Environment variables
          shellHook = ''
            export RUST_SRC_PATH="${pkgs.rust.packages.stable.rustPlatform.rustLibSrc}"
          '';
        };
      }
    );
  };
}
```

### 4. Testing

Implement comprehensive testing:

```nix
{
  outputs = { self, nixpkgs }: {
    checks = forAllSystems (system:
      let
        pkgs = nixpkgs.legacyPackages.${system};
      in {
        # Unit tests
        unit-tests = pkgs.runCommand "unit-tests" {
          buildInputs = [ pkgs.python3 ];
        } ''
          python -m unittest discover ./tests
          touch $out
        '';
        
        # Integration tests
        integration-tests = pkgs.nixosTest {
          name = "integration";
          nodes.machine = { ... }: {
            imports = [ self.nixosModules.default ];
            services.myservice.enable = true;
          };
          
          testScript = ''
            machine.start()
            machine.wait_for_unit("myservice")
            machine.succeed("curl localhost:8080")
          '';
        };
      }
    );
  };
}
```

### 5. Documentation

Document your flake effectively:

```nix
{
  outputs = { self, nixpkgs }: {
    # Generate documentation
    docs = forAllSystems (system:
      let
        pkgs = nixpkgs.legacyPackages.${system};
      in
      pkgs.stdenv.mkDerivation {
        name = "project-docs";
        src = ./.;
        
        buildInputs = [ pkgs.mdbook ];
        
        buildPhase = ''
          mdbook build
        '';
        
        installPhase = ''
          cp -r book $out
        '';
      }
    );
  };
}
```

## Advanced Patterns

### 1. Cross-Compilation

Handle cross-compilation elegantly:

```nix
{
  outputs = { self, nixpkgs }: {
    packages = forAllSystems (system:
      let
        pkgs = nixpkgs.legacyPackages.${system};
        
        # Create cross-compilation environments
        crossSystems = {
          aarch64-linux = pkgs.pkgsCross.aarch64-multiplatform;
          riscv64-linux = pkgs.pkgsCross.riscv64;
        };
        
        # Build package for all targets
        mkCrossPackage = target: pkg:
          (crossSystems.${target}.callPackage pkg {});
      in {
        # Regular package
        default = pkgs.callPackage ./packages/default.nix {};
        
        # Cross-compiled variants
        cross = nixpkgs.lib.mapAttrs mkCrossPackage {
          aarch64-linux = ./packages/default.nix;
          riscv64-linux = ./packages/default.nix;
        };
      }
    );
  };
}
```

### 2. Cache Management

Optimize binary cache usage:

```nix
{
  outputs = { self, nixpkgs }: {
    nixConfig = {
      # Specify binary caches
      substituters = [
        "https://cache.nixos.org"
        "https://mycache.org"
      ];
      
      # Trusted public keys
      trusted-public-keys = [
        "cache.nixos.org-1:6NCHdD59X431o0gWypbMrAURkbJ16ZPMQFGspcDShjY="
        "mycache.org:${builtins.readFile ./keys/cache-pub-key}"
      ];
      
      # Cache settings
      keep-outputs = true;
      keep-derivations = true;
    };
  };
}
```

## Best Practices Summary

1. **Modularity**
   - Split configurations into logical modules
   - Use overlays for package customization
   - Create reusable library functions

2. **Reproducibility**
   - Pin input versions
   - Use `flake.lock` for deterministic builds
   - Document all dependencies

3. **Testing**
   - Include unit and integration tests
   - Test across multiple platforms
   - Verify reproducibility

4. **Documentation**
   - Document all options and configurations
   - Include examples
   - Maintain a changelog

5. **Performance**
   - Optimize cache usage
   - Use binary caches
   - Implement proper garbage collection

## Resources

- [Nix Manual](https://nixos.org/manual/nix/stable/)
- [NixOS Manual](https://nixos.org/manual/nixos/stable/)
- [Nix Flakes](https://nixos.wiki/wiki/Flakes) 