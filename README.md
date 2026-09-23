# Architecture Verification

This repository contains the UPPAAL models and supporting artefacts for the availability verification case study presented in the paper.

## Verification

The **baseline** and **refined** UPPAAL models can be opened and tested using [UPPAAL](https://uppaal.org/). After opening a model, verification queries can be executed from the **Verifier** tab.

The repository contains the following artefacts:

* `PastaFactory_AvailabilityModel_Baseline.xml` - baseline UPPAAL model.
* `PastaFactory_AvailabilityModel_Refined.xml` - refined UPPAAL model.
* `AV7_trace.txt` - verification trace illustrating the failure of availability requirement AV-7 in the baseline model.
* `REQUIREMENTS.md` - overview of the case study requirements, quality attributes, and related information.
