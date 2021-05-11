# MycoSNP: GATK Variants

## Overview

This repository contains the MycoSNP GATK Variants workflow, which consists of 10 steps:

1. Call variants using the GATK 4.1.4.1 HaplotypeCaller tool.
2. Combine gVCF files from the HaplotypeCaller into a single VCF using the GATK 4.1.4.1 CombineGVCFs tool.
3. Call genotypes using the GATK 4.1.4.1 GenotypeGVCFs tool.
4. Filter the variants using the GATK 4.1.4.1 VariantFiltration tool and the default (but customizable) filter: 'QD < 2.0 || FS > 60.0 || MQ < 40.0 || DP < 10'.
5. Run a customized VCF filtering script provided by the Broad Institute.
6. Split the filtered VCF file by sample. 
7. Select only SNPs from the VCF files using the GATK 4.1.4.1 SelectVariants tool.
8. Split the VCF file with SNPs by sample.
9. Create a consensus sequence for each sample using BCFTools 1.9 and SeqTK 1.2.
10. Create a multi-fasta file from the VCF SNP positions using a custom script from Broad.

## Requirements

Before installing and running this workflow, follow the instructions in the main repository to install the requirements: https://github.com/CDCgov/mycosnp

## Installation

First install GeneFlow and its dependencies as follows:

1. Activate the [previously installed](https://github.com/CDCgov/mycosnp) Python virtual environment.

    ```bash
    cd ~/mycosnp
    source gfpy/bin/activate
    ```

2. Clone and install the mycosnp-gatk-variants workflow.

    ```bash
    gf install-workflow --make-apps -g https://github.com/CDCgov/mycosnp-gatk-variants mycosnp-gatk-variants
    ```

    The workflow should now be installed in the `~/mycosnp/mycosnp-gatk-variants` directory.

## Execution

View the workflow parameter requirements using GeneFlow's `help` command:

```
cd ~/mycosnp
gf help mycosnp-gatk-variants
```

Execute the workflow with a command similar to the following. Be sure to replace the `/path/to/indexed/reference` with your indexed reference created using the MycoSNP BWA Reference workflow, and replace the `/path/to/bam/index` with the path to the indexed BAM files created by the MycoSNP BWA Pre-Process workflow:

```
gf --log-level debug run mycosnp-gatk-variants \
    -o ./output \
    -n test-mycosnp-gatk-variants \
    --in.input_folder /path/to/bam/index \
    --in.reference_sequence /path/to/indexed/reference \
    --param.max_perc_amb_samples 10 \
    --param.pair_hmm_threads 8
```

Alternatively, to execute the workflow on an HPC system, you must first set the DRMAA library path environment variable. For example:

```
export DRMAA_LIBRARY_PATH=/opt/sge/lib/lx-amd64/libdrmaa2.so
```

Note that the DRMAA library for your specific scheduler (either UGE/SGE or SLURM) must be installed, and the installed library path may be different in your environment. Once the environment has been configured, execute the workflow as follows:

```
gf --log-level debug run mycosnp-gatk-variants \
    -o ./output \
    -n test-mycosnp-gatk-variants \
    --in.input_folder /path/to/bam/index \
    --in.reference_sequence /path/to/indexed/reference \
    --param.max_perc_amb_samples 10 \
    --param.pair_hmm_threads 8 \
    --ec default:slurm \
    --ep \
        default.slots:8 \
        'default.init:echo `hostname` && mkdir -p $HOME/tmp && export TMPDIR=$HOME/tmp && export _JAVA_OPTIONS=-Djava.io.tmpdir=$HOME/tmp && export XDG_RUNTIME_DIR=' \
```

Arguments are explained below:

1. ``-o ./output``: The workflow's output will be placed in the ``./output`` folder. This folder will be created if it doesn't already exist. 
2. ``-n test-mycosnp-gatk-variants``: This is the name of the workflow job. A sub-folder with the name ``test-mycosnp-gatk-variants`` will be created in ``./output`` for the workflow output. 
3. ``--in.input_folder``: This must point to an output folder created by the MycoSNP BWA Pre-Process workflow, and should contain a sub-folder for each sample. Each sub-folder must contain a BAM file and a BAM index file (.bai)
4. ``--in.reference_sequence``: This must point to an output folder created by the MycoSNP BWA Reference workflow, and should contain a reference sequence (FASTA), a .dict file, and a refererence sequence index (.fai)
5. ``--param.max_perc_amb_samples``: Maximum percent of ambiguous samples allowed for inclusion of SNP in FASTA.
6. ``--param.pair_hmm_threads 8``: Number of threads to use for GATK's variant calling algorithm. Recommended threads is 8.
7. ``--ec default:slurm``: This is the workflow "execution context", which specifies where the workflow will be executed. "gridengine" or "slurm" is recommended, as this will execute the workflow on the HPC. However, "local" may also be used. 
8. ``--ep``: This specifies one or more workflow "execution parameters".
   a. ``default.slots:8``: This specifies the number of CPUs or "slots" to request from the gridengine HPC when executing the workflow. The recommended number of slots is 8, and should match the pair_hmm_threads parameter.
   b. ``'default.init:echo `hostname` && mkdir -p $HOME/tmp && export TMPDIR=$HOME/tmp && export _JAVA_OPTIONS=-Djava.io.tmpdir=$HOME/tmp && export XDG_RUNTIME_DIR='``: This specifies a number of commands to execute on each HPC node to prepare that node for execution. This commands ensure that a local "tmp" directory is used (rather than /tmp), and also resets an environment variable that may interfere with correct execution of singularity containers.

After successful execution, the output directory should contain the following structure:

```
├── vcf-filter
│   ├── vcf-filter
│   └── _log
├── split-vcf-broad
│   ├── split-vcf-broad
│   └── _log
├── gatk-selectvariants
│   ├── gatk-selectvariants
│   └── _log
├── split-vcf-selectvariants
│   ├── split-vcf-selectvariants
│   └── _log
├── consensus
│   ├── consensus
│   └── _log
└── vcf-to-fasta
    ├── _log
    └── vcf-to-fasta
```

The ``consensus`` folder contains each consensus sequence FASTA in separate files. The ``vcf-to-fasta`` folder contains a multi-fasta file with only SNP positions. The ``gatk-selectvariants`` folder contains a VCF file with all samples, and only SNP positions. And the ``vcf-filter`` folder contains a VCF file filtered using a custom script provided by Broad. The ``split-vcf-*`` folders contain VCF files split by sample for each of the filtering steps. 

## Public Domain Standard Notice
This repository constitutes a work of the United States Government and is not
subject to domestic copyright protection under 17 USC § 105. This repository is in
the public domain within the United States, and copyright and related rights in
the work worldwide are waived through the [CC0 1.0 Universal public domain dedication](https://creativecommons.org/publicdomain/zero/1.0/).
All contributions to this repository will be released under the CC0 dedication. By
submitting a pull request you are agreeing to comply with this waiver of
copyright interest.

## License Standard Notice
The repository utilizes code licensed under the terms of the Apache Software
License and therefore is licensed under ASL v2 or later.

This source code in this repository is free: you can redistribute it and/or modify it under
the terms of the Apache Software License version 2, or (at your option) any
later version.

This source code in this repository is distributed in the hope that it will be useful, but WITHOUT ANY
WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE. See the Apache Software License for more details.

You should have received a copy of the Apache Software License along with this
program. If not, see http://www.apache.org/licenses/LICENSE-2.0.html

The source code forked from other open source projects will inherit its license.

## Privacy Standard Notice
This repository contains only non-sensitive, publicly available data and
information. All material and community participation is covered by the
[Disclaimer](https://github.com/CDCgov/template/blob/master/DISCLAIMER.md)
and [Code of Conduct](https://github.com/CDCgov/template/blob/master/code-of-conduct.md).
For more information about CDC's privacy policy, please visit [http://www.cdc.gov/other/privacy.html](https://www.cdc.gov/other/privacy.html).

## Contributing Standard Notice
Anyone is encouraged to contribute to the repository by [forking](https://help.github.com/articles/fork-a-repo)
and submitting a pull request. (If you are new to GitHub, you might start with a
[basic tutorial](https://help.github.com/articles/set-up-git).) By contributing
to this project, you grant a world-wide, royalty-free, perpetual, irrevocable,
non-exclusive, transferable license to all users under the terms of the
[Apache Software License v2](http://www.apache.org/licenses/LICENSE-2.0.html) or
later.

All comments, messages, pull requests, and other submissions received through
CDC including this GitHub page may be subject to applicable federal law, including but not limited to the Federal Records Act, and may be archived. Learn more at [http://www.cdc.gov/other/privacy.html](http://www.cdc.gov/other/privacy.html).

## Records Management Standard Notice
This repository is not a source of government records, but is a copy to increase
collaboration and collaborative potential. All government records will be
published through the [CDC web site](http://www.cdc.gov).

## Additional Standard Notices
Please refer to [CDC's Template Repository](https://github.com/CDCgov/template)
for more information about [contributing to this repository](https://github.com/CDCgov/template/blob/master/CONTRIBUTING.md),
[public domain notices and disclaimers](https://github.com/CDCgov/template/blob/master/DISCLAIMER.md),
and [code of conduct](https://github.com/CDCgov/template/blob/master/code-of-conduct.md).
