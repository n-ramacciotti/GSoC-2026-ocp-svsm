# GSoC 2026 Project Report
- Organization: [QEMU](https://www.qemu.org/)
- Community: [COCONUT-SVSM](https://github.com/coconut-svsm)
- Project: [Observability Support for COCONUT-SVSM](https://summerofcode.withgoogle.com/programs/2026/projects/1Q3qWHfa)
- Contributor: Nicola Ramacciotti ([n-ramacciotti](https://github.com/n-ramacciotti))
- Mentors: Stefano Garzarella ([stefano-garzarella](https://github.com/stefano-garzarella)), Gerd Hoffmann ([kraxel](https://github.com/kraxel)), Joerg Roedel ([joergroedel](https://github.com/joergroedel))

## Project Description
A guest operating system (OS), running inside a Confidential Virtual Machine (CVM), does not know the state information, such as log data or memory usage, of the secure module [COCONUT-SVSM](https://github.com/coconut-svsm/svsm), which is responsible for managing the CVM. These sources of information could be useful for the guest OS to understand the environment it is running in.

A new protocol, called Observability and Configuration Protocol (OCP), has been defined to enable information to be retrieved from an SVSM source, and to allow the configuration of sources in SVSM. All the sources and source categories available in SVSM can be listed using this protocol, and each of them can be read and potentially written.

This project implements the protocol and is divided into three parts:
  -  The implementation of a SVSM-side handler for the protocol.
  -  A guest-side handler in the form of a Linux device driver
  -  A user-space tool that uses the Linux device driver to interact with the sources.

## Contributions

### OCP SVSM-side Implementation PRs

- [kernel/protocols: Reserve OCP number - #1182](https://github.com/coconut-svsm/svsm/pull/1182)  [**open**]

  Different SVSM protocols have different identifiers. This PR simply adds the OCP number to the SVSM codebase.

- [kernel/protocols: Introduce OCP support - #1138](https://github.com/coconut-svsm/svsm/pull/1138) [**open**]

  This PR introduces support for the new protocol with associated documentation. It adds a missing protocol register to enable data exchange between the guest OS and SVSM, and makes the protocol discoverable via the SVSM CORE QUERY protocol. The protocol is also gated behind a Rust feature.

  New structures have been added. These map the objects and sources available in SVSM. New request handlers have also been introduced to list a subset of objects and sources and to read and write a precise number of bytes to a specific source at a specific offset.

  Additionally, some sources were introduced:

  - SVSM version: read only source that gives back the current running SVSM version
  - Log buffer: read only source that contains runtime information of the SVSM

  Since some sources, such as log data, are file-based, a new utility structure was introduced to facilitate the transfer of data from a file to guest memory and vice versa.

### OCP Guest Kernel-side Implementation PRs

- [Add support for OCP protocol - #20](https://github.com/coconut-svsm/linux/pull/20) [**open**]

  New kernel APIs have been introduced to enable interaction with the new SVSM protocol.

  A new not hot-pluggable platform device has been added and registered at boot time for a confidential SEV-SNP environment.

  The APIs can be used with the new device driver, which is only registered when the platform device is available. The driver can be accessed from userspace via the associated character device. The driver can be controlled using the IOCTL interface, which is shared in the `uapi` folder and contains both ioctl commands and structures.

  **note**: Currently, a pull request has been opened against the Linux fork of the Coconut Community. The ultimate goal is to integrate the driver into the main Linux code base.

### OCP Guest Userspace-side Implementation PRs

- [Add first version of svsm-tools - #1](https://github.com/n-ramacciotti/svsm-tools/pull/1)  [**open**]

  This new userspace tool interacts with the new driver via ioctl commands. The `uapi` kernel interface was used to automatically generate a Rust-compatible interface with `bindgen`, which is used together with the `nix` crate to implement the ioctl commands.

  As the kernel does not use sources and category details directly, they are not included in the `uapi` interface. New structures that mirror those available in SVSM, alongside new methods to access their fields, were introduced.

  A new protocol handler has been introduced. This wraps the ioctl commands in order to return the full list of categories or sources and to enable reading from or writing to an entire source. The command line interface parses the argument with the `clap` crate before invoking the corresponding handler method.

### Reviews:
- [Implement a log buffer to store log messages. - #52](https://github.com/coconut-svsm/svsm/pull/52) [**merged**]

  This PR introduces a logging mechanism that stores information in a file instead of printing them to the console, as that in a production environment could be a security problem. This is a necessary observability source for the OCP and I helped review the PR.

### Additional PRs

Contributions, that were not directly related to the new protocol, were also made during this period:

- [scripts/check-signed-off: Avoid exiting on the first failure - #1098](https://github.com/coconut-svsm/svsm/pull/1098) [**merged**]

  Updated a script so that it now checks that all the commits contain `Signed-off-by` instead of exiting on the first miss.

- [kernel/tests: fix test_fpu_context_switch - #1115](https://github.com/coconut-svsm/svsm/pull/1115) [**merged**]

  A missing wait API was introduced after task creation to preserve the correct scheduling order during testing.

- [kernel/task: save exit code - #1128](https://github.com/coconut-svsm/svsm/pull/1128) [**merged**]

  SVSM lacked the ability to store the exit code of a task after its termination. This PR introduced a new `exit_status` field to the `Task` structure, along with new get and set methods. Particular care was taken in the handling of both normal exits and exceptions, with the introduction of a new enum `TaskExitStatus`.

- [kernel/protocols: add vtpm to core query protocol - #1135](https://github.com/coconut-svsm/svsm/pull/1135) [**merged**]

  The guest uses the SVSM CORE QUERY protocol call to check the availability of a particular protocol. The SVSM handler lacked a check for VTPM availability. This PR introduced it.

- [xbuild/fs: Handle multi-level path - #1141](https://github.com/coconut-svsm/svsm/pull/1141) [**merged**]

  There was an issue with the SVSM build system on multi-level paths during file system generation, as intermediate directories were not checked for their existence. This PR resolved this issue by verifying the presence of all directories and creating them if necessary.

- [README.md: Squash duplicate documentation sections - #1158](https://github.com/coconut-svsm/svsm/pull/1158) [**merged**]

  The README.md file previously contained two documentation sections, which have now been merged into one.

- [xbuild/test: Replace dashes in package name - #1160](https://github.com/coconut-svsm/svsm/pull/1160) [**merged**]

  In certain scenarios, Cargo produces an output that slightly differ from the lib name, changing `-` in `_`. I updated the build system to take this into account.

- [userspace: remove direct console access - #1165](https://github.com/coconut-svsm/svsm/pull/1165) [**merged**]

  Even when the kernel was configured to avoid console printing, userspace tasks had a mechanism to access console logging. I removed this direct access, leaving the decision exclusively to the kernel.

- [userspace: Introduce log crate - #1166](https://github.com/coconut-svsm/svsm/pull/1166) [**merged**]

  The `log` trait from the log crate has been implemented, meaning that userspace tasks can now use it. I also removed some redundant or broken variables.

- [userspace: Introduce test in svsm - #926](https://github.com/coconut-svsm/svsm/pull/926) [**merged**]

  Introduced custom test runner for userspace tasks, now it is possible to test the kernel `uapi` at runtime.

- [kernel/guestmem: Support references in slice index writes - #1197](https://github.com/coconut-svsm/svsm/pull/1197) [**merged**]

  The infrastructure for accessing memory outside the SVSM kernel (SVSM user/Guest memory) could access a slice of elements, but could only write to an index through an owned value, not through a reference. This PR adds reference support for this type of access.

## Try it yourself

**note**: It requires AMD Secure Encrypted Virtualization with Secure Nested Paging (AMD SEV-SNP).

### Setup
- Follow the instructions in [INSTALL.md](https://github.com/coconut-svsm/svsm/blob/main/Documentation/docs/installation/INSTALL.md) to compile SVSM. Make sure to use a version of SVSM that is OCP-capable. As previously mentioned, the patch is not yet available upstream and can be found at [#1138](https://github.com/coconut-svsm/svsm/pull/1138)

- Prepare a guest image with an OCP-capable kernel. This patch is also not yet available upstream and can be found at [#20](https://github.com/coconut-svsm/linux/pull/20). Enable `CONFIG_OCP_SVSM` before compilation in the `.config` file. Once the image is ready, launch a virtual machine by following the instructions in [Launch Script](https://github.com/coconut-svsm/svsm/blob/main/Documentation/docs/installation/INSTALL.md#launch-script). Make sure that the `ocp-svsm` module is loaded. You can now interact with SVSM through `/dev/ocp`. Change its permissions so that you can interact without `sudo`.

- In the running VM, clone and build the userspace utility, available at [PR #1](https://github.com/n-ramacciotti/svsm-tools/pull/1).

### Userspace utility

Build and launch the utility with `sudo cargo run -- ...`. Specify the `-h` or `--help` option to view a brief description of each command.

#### Commands
- `list`: displays a list of all sources, grouped by category.
- `read`: reads a specified source by name.
- `write`: writes a specified source by name.

## Future Work
- Some PRs are still open, particularly those related to the OCP. This is normal, as major features require time for testing and review. Additionally, the protocol specifications are not definitive and may change. The next steps require the integration of the open PRs with changes based on the feedback of the reviewers and on the fully defined protocol.

- Currently, only a limited number of sources and categories are available. While this is sufficient for testing the environment, SVSM should be able to offer more observability and configuration sources. The next steps require the addition of more sources and categories.

## Acknowledgements
I would like to thank my mentors for their support throughout the project. I really enjoyed working on it.
