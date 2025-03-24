# Speakeasy Emulator Service for Assemblyline

This repository contains an Assemblyline v4 service that utilizes Mandiant's Speakeasy emulator to analyze Windows executables and shellcode for triage purposes within the Assemblyline framework.

## Overview

The `SpeakeasyEmulator` service is designed to emulate Windows PE files (x86 and amd64 only) and position-independent shellcode through the Speakeasy library.

## Triage Indicators Identified

The service identifies the following indicators based on the Speakeasy report:

* **Multiple TLS Callbacks:** Flags PE files with more than one TLS (Thread Local Storage) callback, which may indicate evasion techniques.
* **`VirtualAlloc` with `PAGE_EXECUTE_READWRITE`:** Detects the allocation of memory with both execute and write permissions, common in code injection techniques.
* **Loading Potentially Suspicious DLLs:** Identifies the loading of DLLs frequently associated with malicious activities (e.g., `SHELL32.dll`, `ADVAPI32.dll`, `ntdll.dll`).
* **High Number of `GetProcAddress` Calls:** Flags PE files making a large number of calls to `GetProcAddress`, especially in the second half of execution, which may indicate API resolution after unpacking/decryption.
* **Calls to Suspicious API Functions:** Detects the use of specific Windows API functions known for malicious use (e.g., `CreateRemoteThread`, network, registry operations).
* **Dynamic Code Segments:** Identifies the creation or modification of code segments during runtime, often used to evade static analysis.
* **Shellcode Detection:** Identifies and emulates raw shellcode blobs.
* **Network Activity Detection:** Identifies network communication attempts during emulation, with special attention to shellcode-based networking.

These indicators are presented as heuristics within the Assemblyline analysis results.

## Service Details

The core logic resides in `SpeakeasyEmulator.py`, inheriting from Assemblyline's `ServiceBase`. The `execute` function orchestrates the loading, running, and analysis of the PE file or shellcode using Speakeasy, ultimately generating Assemblyline `ResultSection` objects based on the triage indicators.

## Heuristics (Defined in `service.yml`)

| Heuristic ID | Name                                        | Score | Filetype           | Description                                                                                                                                   |
|:-------------|:--------------------------------------------|:------|:-------------------|:----------------------------------------------------------------------------------------------------------------------------------------------|
| 1            | Multiple TLS Callbacks Detected             | 100   | executable/windows | The PE file contains more than one TLS callbacks. Multiple TLS callbacks are rare in legitimate software but common in malware.               |
| 2            | VirtualAlloc with Execute/Write Permissions | 200   | executable/windows | The PE file allocates memory with both execute and write permissions. Malware often allocates memory with execute permissions to inject code. |
| 3            | Loads Potentially Suspicious DLL            | 300   | executable/windows | The PE file loads a DLL using either 'LoadLibraryA' or 'LoadLibraryW' that is specified in the SUS_DLLS configuration variable.               |
| 4            | High Number of GetProcAddress Calls         | 400   | executable/windows | The PE file makes excessive calls to GetProcAddress in the second half of execution. Often used after unpacking/decryption.                   |
| 5            | Calls Suspicious API Function               | 300   | executable/windows | The PE file calls an API function that is specified in the SUS_APIS configuration variable.                                                   |
| 6            | Dynamic Code Segments Detected              | 130   | executable/windows | The PE file creates or modifies code segments at runtime. Malware often generates code at runtime to evade static analysis.                   |
| 7            | Shellcode Detected                          | 1000  | *                  | File is a shellcode blob and successfully emulated.                                                                                           |
| 8            | Shellcode Network Activity Detected         | 1000  | *                  | Shellcode successfully performed network activity.                                                                                            |

## Requirements

* An operational Assemblyline v4 instance.
* Docker environment configured for Assemblyline service execution.
* The Speakeasy Python library must be installed within the service's Docker container.

## Installation

To use this service in your Assemblyline environment:

1. Place the contents of this repository (specifically the `SpeakeasyEmulator.py` and `service.yml` files) into your Assemblyline's services directory.
2. Ensure that the `Dockerfile` for your Assemblyline service build process includes the installation of the `speakeasy` Python library (e.g., using `pip install speakeasy`).
3. Build and deploy your Assemblyline services, including this Speakeasy emulator service, according to your Assemblyline deployment procedures.

## Usage

### Within Assemblyline

Once the service is successfully deployed and enabled in your Assemblyline instance:

1. Submit a Windows executable file (PE file) or raw PIE shellcode files for analysis through the Assemblyline web interface or API.
2. If enabled, the "Speakeasy" service will process the submitted file during the "CORE" stage of analysis.
3. The analysis results will include a section titled "PE File Triage Indicators (Speakeasy)" if any of the defined triage indicators are detected. This section will contain details about the findings and will trigger the corresponding Assemblyline heuristics, contributing to the overall verdict and score.
4. A raw JSON report generated by Speakeasy will be available as a supplementary file for more in-depth analysis.

### Standalone

You can use the `assemblyline_v4_service` package's `run_service_once` module to locally run the service like below:

```bash
python3 -m assemblyline_v4_service.dev.run_service_once speakeasyEmulator.SpeakeasyEmulator <PE file or shellcode>
```

## Configuration

Service-specific configurations can be managed through the Assemblyline web interface or by directly modifying the `service.yml` file before deployment. Key configuration options include:

* **`SUS_APIS`**: A list of Windows API function names that the service will flag as suspicious if called during emulation. The default list includes APIs related to process creation, network communication, registry operations, and anti-debugging techniques.
* **`SUS_DLLS`**: A list of DLL file names that the service will flag as potentially suspicious if loaded during emulation.
* **`GETPROCADDR_THRESHOLD`**: An integer value representing the minimum number of `GetProcAddress` calls required to trigger the corresponding heuristic. Default is 15.
* **`EMULATE_ALL_ENTRYPOINTS`**: Boolean flag to determine whether to emulate all entrypoints and exports or just the main entrypoint. Default is true.
* **`EMULATE_CHILDREN`**: Boolean flag to determine whether to emulate child processes created by the executable. Default is true.
* **`MAX_FILE_SIZE`**: Maximum file size in bytes that the service will analyze. Default is 52428800 (50MB).
* **`SHELLCODE_ARCH`**: Architecture to emulate for shellcode analysis. Supported values are "amd64" and "x86". Default is "amd64".

## Docker Configuration

The service runs in a Docker container with the following specifications:
* Image: `${REGISTRY}testing/assemblyline-service-speakeasy:$SERVICE_TAG`
* CPU Cores: 1.0
* RAM: 1024 MB
* Timeout: 480 seconds