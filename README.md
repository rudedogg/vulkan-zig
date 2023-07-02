# vulkan-zig

A Vulkan binding generator for Zig.

[![Actions Status](https://github.com/Snektron/vulkan-zig/workflows/Build/badge.svg)](https://github.com/Snektron/vulkan-zig/actions)

## Overview

vulkan-zig attempts to provide a better experience to programming Vulkan applications in Zig, by providing features such as integration of vulkan errors with Zig's error system, function pointer loading, renaming fields to standard Zig style, better bitfield handling, turning out parameters into return values and more.

vulkan-zig is automatically tested daily against the latest vk.xml and zig, and supports vk.xml from version 1.x.163.

### Zig versions

vulkan-zig aims to be always compatible with the ever-changing Zig master branch (however, development may lag a few days behind). Sometimes, the Zig master branch breaks a bunch of functionality however, which may make the latest version vulkan-zig incompatible with older releases of Zig. This repository aims to have a version compatible for both the latest Zig master, and the latest Zig release. The `master` branch is compatible with the `master` branch of Zig, and versions for older versions of Zig are maintained in the `zig-<version>-compat` branch.

`master` is compatible and tested with the Zig self-hosted compiler. The `zig-stage1-compat` branch contains a version which is compatible with the Zig stage 1 compiler.

## Features
### CLI-interface
A CLI-interface is provided to generate vk.zig from the [Vulkan XML registry](https://github.com/KhronosGroup/Vulkan-Docs/blob/main/xml), which is built by default when invoking `zig build` in the project root. To generate vk.zig, simply invoke the program as follows:
```
$ zig-out/bin/vulkan-zig-generator path/to/vk.xml output/path/to/vk.zig
```
This reads the xml file, parses its contents, renders the Vulkan bindings, and formats file, before writing the result to the output path. While the intended usage of vulkan-zig is through direct generation from build.zig (see below), the CLI-interface can be used for one-off generation and vendoring the result.

### Generation from build.zig
Vulkan bindings can be generated from the Vulkan XML registry at compile time with build.zig, by using the provided Vulkan generation step:
```zig
const vkgen = @import("vulkan-zig/generator/index.zig");

pub fn build(b: *Builder) void {
    ...
    const exe = b.addExecutable("my-executable", "src/main.zig");

    // Create a step that generates vk.zig (stored in zig-cache) from the provided vulkan registry.
    const gen = vkgen.VkGenerateStep.create(b, "path/to/vk.xml");

    // Add the generated file as package to the final executable
    exe.addModule("vulkan", gen.getModule());
}
```
This reads vk.xml, parses its contents, and renders the Vulkan bindings to "vk.zig", which is then formatted and placed in `zig-cache`. The resulting file can then be added to an executable by using `addModule`, after which the bindings will be made available to the executable under the name passed to `getModule`.

### Generation with the package manager from build.zig
There is also support for adding this project as a dependency through zig package manager in its current form. In order to do this, add this repo as a dependency in your build.zig.zon:
```zig
.{
    // -- snip --
    .dependencies = .{
        // -- snip --
        .vulkan_zig = .{
            .url = "https://github.com/Snektron/vulkan-zig/archive/<commit SHA>.tar.gz",
            .hash = "<dependency hash>",
        },
    },
}
```
And then in your build.zig file, you'll need to add a line like this to your build function:
```zig
const vkzig_dep = b.dependency("vulkan_zig", .{
    .registry = @as([]const u8, b.pathFromRoot("path/to/vk.xml")),
});
const vkzig_bindings = vkzig_dep.module("vulkan-zig");
exe.addModule("vulkan-zig", vkzig_bindings);
```
That will allow you to `@import("vulkan-zig")` in your executable's source.

### Manual generation with the package manager from build.zig
In the event you have a specific need for it, the generator executable is made available through the dependency, allowing you to run the executable as a build step in your own build.zig file.
Doing so should look a bit like this:
```zig
const vk_gen = b.dependency("vulkan_zig", .{}).artifact("generator"); // get generator executable reference

const generate_cmd = b.addRunArtifact(vk_gen);
generate_cmd.addArg(b.pathFromRoot("vk.xml")); // path to xml file to use when generating the bindings

const vulkan_zig = b.addModule("vulkan-zig", .{
    .source_file = generate_cmd.addOutputFileArg("vk.zig"), // this is the FileSource representing the generated bindings
});
exe.addModule("vulkan-zig", vulkan_zig);
```

### Function & field renaming
Functions and fields are renamed to be more or less in line with [Zig's standard library style](https://ziglang.org/documentation/master/#Style-Guide):
* The vk prefix is removed everywhere
  * Structs like `VkInstanceCreateInfo` are renamed to `InstanceCreateInfo`.
  * Handles like `VkSwapchainKHR` are renamed to `SwapchainKHR` (note that the tag is retained in caps).
  * Functions like `vkCreateInstance` are generated as `createInstance` as wrapper and as `PfnCreateInstance` as function pointer.
  * API constants like `VK_WHOLE_SIZE` retain screaming snake case, and are generates as `WHOLE_SIZE`.
* The type name is stripped from enumeration fields and bitflags, and they are generated in (lower) snake case. For example, `VK_IMAGE_LAYOUT_GENERAL` is generated as just `general`. Note that author tags are also generated to lower case: `VK_SURFACE_TRANSFORM_FLAGS_IDENTITY_BIT_KHR` is translated to `identity_bit_khr`.
* Container fields and function parameter names are generated in (lower) snake case in a similar manner: `ppEnabledLayerNames` becomes `pp_enabled_layer_names`.
* Any name which is either an illegal Zig name or a reserved identifier is rendered using `@"name"` syntax. For example, `VK_IMAGE_TYPE_2D` is translated to `@"2d"`.

### Function pointers & Wrappers
vulkan-zig provides no integration for statically linking libvulkan, and these symbols are not generated at all. Instead, vulkan functions are to be loaded dynamically. For each Vulkan function, a function pointer type is generated using the exact parameters and return types as defined by the Vulkan specification:
```zig
pub const PfnCreateInstance = fn (
    p_create_info: *const InstanceCreateInfo,
    p_allocator: ?*const AllocationCallbacks,
    p_instance: *Instance,
) callconv(vulkan_call_conv) Result;
```

For each function, a wrapper is generated into one of three structs:
* BaseWrapper. This contains wrappers for functions which are loaded by `vkGetInstanceProcAddr` without an instance, such as `vkCreateInstance`, `vkEnumerateInstanceVersion`, etc.
* InstanceWrapper. This contains wrappers for functions which are otherwise loaded by `vkGetInstanceProcAddr`.
* DeviceWrapper. This contains wrappers for functions which are loaded by `vkGetDeviceProcAddr`.

Each wrapper struct can be called with an array of the appropriate enums:
```zig
const vk = @import("vulkan");
const BaseDispatch = vk.BaseWrapper(.{
    .createInstance = true,
});
```
The wrapper struct then provides wrapper functions for each function pointer in the dispatch struct:
```zig
pub const BaseWrapper(comptime cmds: anytype) type {
    ...
    const Dispatch = CreateDispatchStruct(cmds);
    return struct {
        dispatch: Dispatch,

        pub const CreateInstanceError = error{
            OutOfHostMemory,
            OutOfDeviceMemory,
            InitializationFailed,
            LayerNotPresent,
            ExtensionNotPresent,
            IncompatibleDriver,
            Unknown,
        };
        pub fn createInstance(
            self: Self,
            create_info: InstanceCreateInfo,
            p_allocator: ?*const AllocationCallbacks,
        ) CreateInstanceError!Instance {
            var instance: Instance = undefined;
            const result = self.dispatch.vkCreateInstance(
                &create_info,
                p_allocator,
                &instance,
            );
            switch (result) {
                .success => {},
                .error_out_of_host_memory => return error.OutOfHostMemory,
                .error_out_of_device_memory => return error.OutOfDeviceMemory,
                .error_initialization_failed => return error.InitializationFailed,
                .error_layer_not_present => return error.LayerNotPresent,
                .error_extension_not_present => return error.ExtensionNotPresent,
                .error_incompatible_driver => return error.IncompatibleDriver,
                else => return error.Unknown,
            }
            return instance;
        }

        ...
    }
}
```
Wrappers are generated according to the following rules:
* The return type is determined from the original return type and the parameters.
  * Any non-const, non-optional single-item pointer is interpreted as an out parameter.
  * If a command returns a non-error `VkResult` other than `VK_SUCCESS` it is also returned.
  * If there are multiple return values selected, an additional struct is generated. The original call's return value is called `return_value`, `VkResult` is named `result`, and the out parameters are called the same except `p_` is removed. They are generated in this order.
* Any const non-optional single-item pointer is interpreted as an in-parameter. For these, one level of indirection is removed so that create info structure pointers can now be passed as values, enabling the ability to use struct literals for these parameters.
* Error codes are translated into Zig errors.
* As of yet, there is no specific handling of enumeration style commands or other commands which accept slices.

Furthermore, each wrapper contains a function to load each function pointer member when passed either `PfnGetInstanceProcAddr` or `PfnGetDeviceProcAddr`, which attempts to load each member as function pointer and casts it to the appropriate type. These functions are loaded literally, and any wrongly named member or member with a wrong function pointer type will result in problems.
* For `BaseWrapper`, this function has signature `fn load(loader: anytype) error{CommandFailure}!Self`, where the type of `loader` must resemble `PfnGetInstanceProcAddr` (with optionally having a different calling convention).
* For `InstanceWrapper`, this function has signature `fn load(instance: Instance, loader: anytype) error{CommandFailure}!Self`, where the type of `loader` must resemble `PfnGetInstanceProcAddr`.
* For `DeviceWrapper`, this function has signature `fn load(device: Device, loader: anytype) error{CommandFailure}!Self`, where the type of `loader` must resemble `PfnGetDeviceProcAddr`.

Note that these functions accepts a loader with the signature of `anytype` instead of `PfnGetInstanceProcAddr`. This is because it is valid for `vkGetInstanceProcAddr` to load itself, in which case the returned function is to be called with the vulkan calling convention. This calling convention is not required for loading vulkan-zig itself, though, and a loader to be called with any calling convention with the target architecture may be passed in. This is particularly useful when interacting with C libraries that provide `vkGetInstanceProcAddr`.

```zig
// vkGetInstanceProcAddr as provided by GLFW.
// Note that vk.Instance and vk.PfnVoidFunction are ABI compatible with VkInstance,
// and that `extern` implies the C calling convention.
pub extern fn glfwGetInstanceProcAddress(instance: vk.Instance, procname: [*:0]const u8) vk.PfnVoidFunction;

// Or provide a custom implementation.
// This function is called with the unspecified Zig-internal calling convention.
fn customGetInstanceProcAddress(instance: vk.Instance, procname: [*:0]const u8) vk.PfnVoidFunction {
    ...
}

// Both calls are valid, even
const vkb = try BaseDispatch.load(glfwGetInstanceProcAddress);
const vkb = try BaseDispatch.load(customGetInstanceProcAddress);
```

By default, wrapper `load` functions return `error.CommandLoadFailure` if a call to the loader resulted in `null`. If this behaviour is not desired, one can use `loadNoFail`. This function accepts the same parameters as `load`, but does not return an error any function pointer fails to load and sets its value to `undefined` instead. It is at the programmer's discretion not to invoke invalid functions, which can be tested for by checking whether the required core and extension versions the function requires are supported.

One can access the underlying unwrapped C functions by doing `wrapper.dispatch.vkFuncYouWant(..)`.

### Bitflags
Packed structs of bools are used for bit flags in vulkan-zig, instead of both a `FlagBits` and `Flags` variant. Places where either of these variants are used are both replaced by this packed struct instead. This means that even in places where just one flag would normally be accepted, the packed struct is accepted. The programmer is responsible for only enabling a single bit.

Each bit is defaulted to `false`, and the first `bool` is aligned to guarantee the overal alignment
of each Flags type to guarantee ABI compatibility when passing bitfields through structs:
```zig
pub const QueueFlags = packed struct {
    graphics_bit: bool align(@alignOf(Flags)) = false,
    compute_bit: bool = false,
    transfer_bit: bool = false,
    sparse_binding_bit: bool = false,
    protected_bit: bool = false,
    _reserved_bit_5: bool = false,
    _reserved_bit_6: bool = false,
    ...
}
```
Note that on function call ABI boundaries, this alignment trick is not sufficient. Instead, the flags
are reinterpreted as an integer which is passed instead. Each flags type is augmented by a mixin which provides `IntType`, an integer which represents the flags on function ABI boundaries. This mixin also provides some common set operation on bitflags:
```zig
pub fn FlagsMixin(comptime FlagsType: type) type {
    return struct {
        pub const IntType = Flags;

        // Return the integer representation of these flags
        pub fn toInt(self: FlagsType) IntType {...}

        // Turn an integer representation back into a flags type
        pub fn fromInt(flags: IntType) FlagsType { ... }

        // Return the set-union of `lhs` and `rhs.
        pub fn merge(lhs: FlagsType, rhs: FlagsType) FlagsType { ... }

        // Return the set-intersection of `lhs` and `rhs`.
        pub fn intersect(lhs: FlagsType, rhs: FlagsType) FlagsType { ... }

        // Return the set-complement of `lhs` and `rhs`. Note: this also inverses reserved bits.
        pub fn complement(self: FlagsType) FlagsType { ... }

        // Return the set-subtraction of `lhs` and `rhs`: All fields set in `rhs` are cleared in `lhs`.
        pub fn subtract(lhs: FlagsType, rhs: FlagsType) FlagsType { ... }

        // Returns whether all bits set in `rhs` are also set in `lhs`.
        pub fn contains(lhs: FlagsType, rhs: FlagsType) bool { ... }
    };
}
```

### Handles
Handles are generated to a non-exhaustive enum, backed by a `u64` for non-dispatchable handles and `usize` for dispatchable ones:
```zig
const Instance = extern enum(usize) { null_handle = 0, _ };
```
This means that handles are type-safe even when compiling for a 32-bit target.

### Struct defaults
Defaults are generated for certain fields of structs:
* sType is defaulted to the appropriate value.
* pNext is defaulted to `null`.
* No other fields have default values.
```zig
pub const InstanceCreateInfo = extern struct {
    s_type: StructureType = .instance_create_info,
    p_next: ?*const anyopaque = null,
    flags: InstanceCreateFlags,
    ...
};
```

### Pointer types
Pointer types in both commands (wrapped and function pointers) and struct fields are augmented with the following information, where available in the registry:
* Pointer optional-ness.
* Pointer const-ness.
* Pointer size: Either single-item, null-terminated or many-items.

Note that this information is not everywhere as useful in the registry, leading to places where optional-ness is not correct. Most notably, CreateInfo type structures which take a slice often have the item count marked as optional, but the pointer itself not. As of yet, this is not fixed in vulkan-zig. If drivers properly follow the Vulkan specification, these can be initialized to `undefined`, however, [that is not always the case](https://zeux.io/2019/07/17/serializing-pipeline-cache/).

### Platform types
Defaults with the same ABI layout are generated for most platform-defined types. These can either by bitcasted to, or overridden by defining them in the project root:
```zig
pub const xcb_connection_t = if (@hasDecl(root, "xcb_connection_t")) root.xcb_connection_t else @Type(.Opaque);
```
For some times (such as those from Google Games Platform) no default is known. Usage of these without providing a concrete type in the project root generates a compile error.

### Shader compilation
vulkan-zig provides functionality to help compiling shaders to spir-v using glslc. It can be used from build.zig as follows:

```zig
const vkgen = @import("vulkan-zig/generator/index.zig");

pub fn build(b: *Builder) void {
    ...
    const exe = b.addExecutable("my-executable", "src/main.zig");

    const gen = vkgen.VkGenerateStep(b, "path/to/vk.xml", "vk.zig");
    exe.addPackage(gen.package);

    const shader_comp = vkgen.ShaderCompileStep.create(
        builder,
        &[_][]const u8{"glslc", "--target-env=vulkan1.2"}, // Path to glslc and additional parameters
    );
    exe.addPackage(shader_comp.getPackage("shaders"));
    shader_comp.add("shader", "path/to/shader.frag", .{});
}
```
Upon compilation, glslc is then invoked to compile each shader, and the result is placed within `zig-cache`. All shaders which are compiled using a particular `ShaderCompileStep` are imported in a single Zig file using `@embedFile`, and this file can be added to an executable as a package using `getPackage`. To slightly improve compile times, shader compilation is cached; as long as a shader's source and its compile commands stay the same, the shader is not recompiled. The spir-v code for any particular shader is aligned to that of a 32-bit integer as follows, as required by vkCreateShaderModule:
```zig
pub const ${name} align(@alignOf(u32)) = @embedFile("${path}").*;
```

See [build.zig](build.zig) for a working example.

## Limitations
* vulkan-zig has as of yet no functionality for selecting feature levels and extensions when generating bindings. This is because when an extension is promoted to Vulkan core, its fields and commands are renamed to lose the extensions author tag (for example, VkSemaphoreWaitFlagsKHR was renamed to VkSemaphoreWaitFlags when it was promoted from an extension to Vulkan 1.2 core). This leads to inconsistencies when only items from up to a certain feature level is included, as these promoted items then need to re-gain a tag.

## Example
A partial implementation of https://vulkan-tutorial.com is implemented in [examples/triangle.zig](examples/triangle.zig). This example can be ran by executing `zig build run-triangle` in vulkan-zig's root.

## See also
* Implementation of https://vulkan-tutorial.com using `@cImport`'ed bindings: https://github.com/andrewrk/zig-vulkan-triangle.
* Alternative binding generator: https://github.com/SpexGuy/Zig-Vulkan-Headers
* Zig bindings for GLFW: https://github.com/hexops/mach-glfw
  * With vulkan-zig integration example: https://github.com/hexops/mach-glfw-vulkan-example
