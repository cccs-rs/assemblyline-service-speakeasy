# Speakeasy Emulator Service for Assemblyline

This repository contains an Assemblyline v4 service that utilizes the Mandiant's Speakeasy emulator to analyze Windows
executables for triage purposes within the Assemblyline framework.

## Overview

The `SpeakeasyEmulator` service is designed to:

* **Emulate PE files within Assemblyline:** Leverages the Speakeasy library to run submitted Windows executables in a
  controlled environment managed by Assemblyline.
* **Generate a detailed report:** Speakeasy produces a JSON report containing information about API calls, entry points,
  and other execution details, accessible as a supplementary file in Assemblyline.
* **Identify triage indicators for Assemblyline:** The service analyzes the Speakeasy report and flags specific patterns
  and behaviors often associated with suspicious or malicious files, presenting these findings within Assemblyline's
  result structure.
* **Seamless Assemblyline Integration:** Built using the Assemblyline v4 framework, ensuring smooth operation within an
  Assemblyline pipeline.

## Triage Indicators Identified

The service currently identifies the following indicators based on the Speakeasy report:

* **Multiple TLS Callbacks:** Flags PE files with more than one TLS (Thread Local Storage) callback.
* **`VirtualAlloc` with `PAGE_EXECUTE_READWRITE`:** Detects the allocation of memory with both execute and write
  permissions.
* **Loading Potentially Suspicious DLLs:** Identifies the loading of DLLs frequently associated with malicious
  activities (e.g., `SHELL32.dll`, `ADVAPI32.dll`, `ntdll.dll`).
* **High Number of `GetProcAddress` Calls:** Flags PE files making a large number of calls to `GetProcAddress`.
* **Calls to Suspicious API Functions:** Detects the use of specific Windows API functions known for malicious use (
  e.g., `CreateRemoteThread`, network, registry).
* **Dynamic Code Segments:** Identifies the creation or modification of code segments during runtime.

These indicators are presented as heuristics within the Assemblyline analysis results.

## Service Details

The core logic resides in `SpeakeasyEmulator.py`, inheriting from Assemblyline's `ServiceBase`. The `execute` function
orchestrates the loading, running, and analysis of the PE file using Speakeasy, ultimately generating Assemblyline
`ResultSection` objects based on the triage indicators.

## Heuristics (Defined in `service.yml`)

| Heuristic ID | Name                                        | Score | Description                                                               |
|:-------------|:--------------------------------------------|:------|:--------------------------------------------------------------------------|
| 1            | Multiple TLS Callbacks Detected             | 100   | The PE file contains more than one TLS callback function.                 |
| 2            | VirtualAlloc with Execute/Write Permissions | 150   | The PE file allocates memory with both execute and write permissions.     |
| 3            | Loads Potentially Suspicious DLL            | 75    | The PE file loads a DLL that is often associated with malicious activity. |
| 4            | High Number of GetProcAddress Calls         | 60    | The PE file makes a large number of calls to GetProcAddress.              |
| 5            | Calls Suspicious API Function               | 120   | The PE file calls an API function commonly used in malicious activities.  |
| 6            | Dynamic Code Segments Detected              | 130   | The PE file creates or modifies code segments at runtime.                 |
| 7            | Shellcode detected                          | 1000  | Speakeasy heuristic analysis hit.                                         |

## Requirements

* An operational Assemblyline v4 instance.
* Docker environment configured for Assemblyline service execution.
* The Speakeasy Python library must be installed within the service's Docker container.

## Installation

To use this service in your Assemblyline environment:

1. Place the contents of this repository (specifically the `SpeakeasyEmulator.py` and `service.yml` files) into your
   Assemblyline's services directory.
2. Ensure that the `Dockerfile` for your Assemblyline service build process includes the installation of the `speakeasy`
   Python library (e.g., using `pip install speakeasy`).
3. Build and deploy your Assemblyline services, including this Speakeasy emulator service, according to your
   Assemblyline deployment procedures.

## Usage

### Within Assemblyline

Once the service is successfully deployed and enabled in your Assemblyline instance:

1. Submit a Windows executable file (PE file) for analysis through the Assemblyline web interface or API.
2. If enabled, the "Speakeasy" service will process the submitted file during the "CORE" stage of analysis.
3. The analysis results will include a section titled "PE File Triage Indicators (Speakeasy)" if any of the defined
   triage indicators are detected. This section will contain details about the findings and will trigger the
   corresponding Assemblyline heuristics, contributing to the overall verdict and score.
4. A raw JSON report generated by Speakeasy will be available as a supplementary file for more in-depth analysis.

### Standalone

You can use the `assemblyline_v4_service` package's `run_service_once` module to locally run the service like below:

```bash
python3 - m assemblyline_v4_service.dev.run_service_once speakeasyEmulator.SpeakeasyEmulator < PE file >
```

## Configuration

Service-specific configurations can be managed through the Assemblyline web interface or by directly modifying the
`service.yml` file before deployment. Key configuration options include:

* **`SUS_APIS`**: A list of Windows API function names that the service will flag as suspicious if called during
  emulation. Modify this list in `service.yml` to suit your needs.
* **`SUS_DLLS`**: A list of DLL file names that the service will flag as potentially suspicious if loaded during
  emulation. Customize this list in `service.yml`.
* **`GETPROCADDR_THRESHOLD`**: An integer value representing the minimum number of `GetProcAddress` calls required to
  trigger the corresponding heuristic. Adjust this value in `service.yml`.