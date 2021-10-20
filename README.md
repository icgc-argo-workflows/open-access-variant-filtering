# ICGC-ARGO Open-access Somatic Variant Filtering Workflow

## Introduction
This repository maintains the source code of the ICGC ARGO Open-access Somatic Variant Filtering Workflow. It is a bioinformatics workflow that can be used to filter SNV/InDel VCFs based on predefined genomic regions. 

For ICGC-ARGO, these genomic regions are of our most research interests and the variants among these regions can potentially go to be open-access with minimum risk of germline leakage. Please refer to [ICGC-ARGO Open-access Regions](https://github.com/icgc-argo/open-access-regions) for details of how we define and generate the regions. 

The workflow is built using Nextflow DSLv2, with modules imported from other ICGC ARGO Workflows GitHub repositories. It uses Docker containers making installation trivial and results highly reproducible. Specifically, here are repositories maintaining various tools/modules:
- https://github.com/icgc-argo/data-processing-utility-tools
- https://github.com/icgc-argo/variant-calling-tools/variant-filter
- https://github.com/icgc-argo/nextflow-data-processing-utility-tools

Each Nextflow module (including associated container image which is registered in Quay.io) is strictly version controlled and released independently. To ensure reproducibility the pipeline declares explicitly which specific version of a module is to be imported.


## Workflow steps and summary
- `download`: Download input variant calling metadata and `VCF` from `SONG/SCORE`
- `metadata parse`: Parse the metadata to get original variant calling tool which was used to generate the input `VCF`
- `filter`: Perform `SNV/InDel` variants filtering with `Bcftools:view` using different filters according to various variant calling tools
- `payload generation`: Generate `SONG` metadata for filtered `SNV/InDel` calls and upload them to `SONG/SCORE`



## Usage
### Prerequisite
- Install [Nextflow](https://www.nextflow.io/docs/latest/getstarted.html#installation) (>=20.10). 
- Install [Docker](https://docs.docker.com/engine/install/) for full pipeline reproducibility. 

### Reference
You will need to provide a BED file containing genomic regions about the VCFs you would like to filter before running the workflow. Use this parameter to specify its location. 
```
--regions_file '[path to genomic region file]'
```

### Running the workflow
The typical command for running the workflow of verion `0.4.0` is as follows:
```
nextflow run icgc-argo-workflows/open-access-variant-filtering -r 0.4.0 -params-file <your_params_file.json>
```
This will launch the workflow with the docker configuration profile. See below for more information about profiles.

Note that the pipeline will create the following files in your working directory:
```
work            # Directory containing the nextflow working files
results         # Output results (configurable by `publish_dir`, see below)
.nextflow_log   # Log file from Nextflow
# Other nextflow hidden files, eg. history of pipeline runs and old logs.
```

The workflow could be setting to run in two modes: `local` mode and `RDPC` mode.

### Local mode
For `local` mode, both the SNV/InDel variant calling VCFs/Index and related metadata need to be in place at local. You will also need to spcify the path to keep the outputs.

- Inputs

```
--analysis_metadata '[path to metadata file]'
--vcf_file '[path to SNV/InDel variant calling VCF]'
--regions_file '[path to local genomic region file]' 
--publish_dir '[path to keep the outputs]'
```

- Outputs

You will find the workflow outputs under the specified `publish_dir` directory. For instance:
```
OpenFilterWf_pGenVar
    ├── 51a7ee15-f114-468c-80f0-a1f5bd88e100.variant_processing.payload.json
    └── out
        ├── TEST-PR.DO250183.SA610229.wgs.20200513.sanger-wgs.somatic.snv.open-filter.vcf.gz
        └── TEST-PR.DO250183.SA610229.wgs.20200513.sanger-wgs.somatic.snv.open-filter.vcf.gz.tbi
```

### RDPC mode
For `RDPC` mode, SONG/SCORE services need to be available and you have appropriate API token. You also need to make sure the referece files have been pre-staged into RDPC NFS.  

- Inputs

```
--study_id '[ICGC-ARGO study ID]'
--analysis_id '[ICGC-ARGO SONG variant_calling analysis ID]'
--song_url": '[ICGC-ARGO SONG server URL]'
--score_url": '[ICGC-ARGO SCORE server URL]'
--regions_file '[path to ICGC-ARGO RDPC NFS genomic region file]' 
```

- Outputs

The generated outputs are uploaded to SONG/SCORE automatically. You will need use the output `analysis_id` to retrieve both the metadata and filtered VCFs. 


## Parameters

**Argument name**|**Description**|**Default value**|**Requirement**
-----|-----|-----|-----
study\_id|ICGC-ARGO study ID|null|Required in RDPC mode
analysis\_id|ICGC-ARGO SONG analysis ID|null|Required in RDPC mode
analysis\_metadata|Local SONG analysis metadata|null|Required in local mode
vcf\_file|Local SNV/InDel variant calling VCF. <br> The index file is expected to be under the same folder|null|Required in local mode
region\_file|BED file to define ARGO open access regions|null|Required
api\_token|SONG/SCORE API token|null|Required in RDPC mode
song\_url|SONG server URL|null|Required in RDPC mode
score\_url|SCORE server URL|null|Required in RDPC mode
output\_type|Output <br> - compressed BCF (b) <br> - uncompressed BCF (u) <br> - compressed VCF (z) <br> - uncompressed VCF (v)|z|Optional
apply\_filters|Skip sites where FILTER column does not contain any of the strings listed in LIST.|  {<br>'CaVEMan': "PASS", <br>'Pindel': "PASS", <br>'GATK:Mutect2': "PASS" <br>} |Optional
include|Select sites for which the expression is true|{<br>'CaVEMan': "INFO/CLPM=0 && INFO/ASRD>=0.93", <br>'Pindel': "", <br>'GATK:Mutect2': ""<br>} |Optional
exclude|Exclude sites for which the expression is true|{<br>'CaVEMan': "", <br>'Pindel': "", <br>'GATK:Mutect2': ""<br>} |Optional
open|If true(default), then the output files are set with open access|true|Optional
cleanup|If true(default), then clean up the inputs/temporary/outputs under the work\_dir. <br> Skip cleanup when running in local mode.|true|Optional 
cpus|Set requirements for number of CPUs for all processes|1|Optional
mem|Set requirements for memory in GB for all processes |1|Optional


## Custom configuration 

### Resource requests
Although the default requirements set within the workflow will hopefully work for most people and with most input data, you may find that you want to customise the compute resources that the workflow requests. Each step in the workflow has a default set of requirements for number of CPUs and memory. You can easily change the default values by setting them in the `params-file`. 

For instance, if you want to change the default settings for `download` step, you can specify parameters for `download` step in the `params-file` as follow:
```
"download": 
  {
    "song_cpus": 2,
    "song_mem": 2,
    "score_cpus": 3,
    "score_mem": 8
  }
```
Nextflow will overwrite the default values of settings that you provide via the provided `params-file`.

### Tool specific options
For the ultimate flexibility, we are using Nextflow DSL2 modules in a way where it is possible for both developers and users to change tool-specific command-line arguments (e.g. using a different filtering params for `filter` step) as well as `publish_dir` options (e.g. saving files generated by the `filter` step to specific folder). In most cases, as a user you won't have to change the default options set by the workflow developer(s), however, there may be some cases where providing a custom params file can improve the behaviour or increase the reusability of the workflow.

For instance, if you want to change the default filtering settings for `filter` step, you can overwrite the defaults by setting param `filter` in the `params-file` as follow:
```
"filter":
  {
    'cpus': 2,
    'mem': 3,
    'publish_dir': 'my_dir',
    'regions_file': 'my_regions_file',
    'apply_filters': {
      'CaVEMan': "VUM", 
      'Pindel': "FF004", 
      'GATK:Mutect2': "strand_bias"
    },
    'include': {
      'CaVEMan': "INFO/CLPM>0 && INFO/ASRD>=0.83", 
      'Pindel': "", 
      'GATK:Mutect2': ""
    },
    'exclude': {
      'CaVEMan': "", 
      'Pindel': "", 
      'GATK:Mutect2': ""
    },
    'output_type': 'b'
  }
```




