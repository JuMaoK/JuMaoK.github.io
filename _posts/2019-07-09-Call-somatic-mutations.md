

---
layout: post
title: "Call somatic mutations"

tags: [GATK, Somatic mutations]

---

体细胞突变分析

* 目录
{:toc .toc}
---

## References

[#11136 Call somatic mutations using GATK4 Mutect2 ](https://software.broadinstitute.org/gatk/documentation/article?id=11136)

[#24057 Call somatic mutations using GATK4 Mutect2](https://software.broadinstitute.org/gatk/documentation/article?id=24057)

本文内容基于GATK4.1.2.0版本





Call somatic SNVs and indels via local assembly of haplotypes

![](https://ae01.alicdn.com/kf/HTB1tls_e8OD3KVjSZFF763n9pXa9.png)



## Call somatic short variants and generate a bamout with Mutect2

Mutect2有3个模式：(i) tumor-normal mode ; (ii) tumor-only mode; (iii) mitochondrial mode

### tumor-normal mode

```sh
gatk Mutect2 \
     -R reference.fa \
     -I tumor1.bam \
     -I tumor2.bam \
     -I normal1.bam \
     -I normal2.bam \
     -normal normal1_sample_name \
     -normal normal2_sample_name \
     --germline-resource af-only-gnomad.vcf.gz \
     --panel-of-normals pon.vcf.gz \
     -O somatic.vcf.gz

# 实际例子
gatk Mutect2 \
-R ref.fa \
-I tumor.bam \
-I normal.bam \
-normal HCC1143_normal \
-pon resources/chr17_m2pon.vcf.gz \
--germline-resource resources/chr17_af-only-gnomad_grch38.vcf.gz \
--disable-read-filter MateOnSameContigOrNoMappedMateReadFilter \
-L chr17plus.interval_list \
-O 1_somatic_m2.vcf.gz \
-bamout 2_tumor_normal_m2.bam
```

- `--germline-resource`: 指定一个代表群体生殖细胞变异位点和频率的文件，用于call体细胞变异前的过滤。人类样本可以用broadinstitute提供的af-only-gnomad.hg38.vcf.gz，非人样本需要自己构建。这个文件与pon都是推荐提供，但缺少了也能跑。如果无法提供，需要指定`--af-of-alleles-not-in-resource `的数值，所有频率低于该数值的变异都会被认为是不可靠的，相当于硬过滤。
- `--disable-read-filter`: 禁用指定的read过滤器，它们往往在分析前对read进行过滤。
- `-L`: 相当于`--intervals`，指定分析只在特定区间进行，上面例子为了节约时间而选用了17号染色体进行测试。
- `-bamout`：File to which assembled haplotypes should be written
- 过滤器**MateOnSameContigOrNoMappedMateReadFilter**被禁用，该过滤器的作用是只保留配对到同一contig的read pairs，或者mate无法被mapped的read（Keep only reads that have a mate that maps to the same contig (RNEXT is "="), is single ended (not 0x1) or has an unmapped mate (0x8).）。这里禁用的原因是提高变异call的数量。



## Create a sites-only PoN with CreateSomaticPanelOfNormals 

Panel of Normal: 在体细胞突变分析中，Mutect2和GetPileupSummaries 都会需要用到这个数据。来源于normal样本，比如在癌细胞分析中即来自健康组织确定没有癌变的体细胞样本，往往用血液样本。broadinstitue有提供标准的pon，但我们也可以用[CreateSomaticPanelOfNormals](https://software.broadinstitute.org/gatk/documentation/tooldocs/current/org_broadinstitute_hellbender_tools_walkers_mutect_CreateSomaticPanelOfNormals.php)自己做一个：

- Step1: 以tumor-only模式运行mutect2，处理每一个正常组的样本：

```sh
gatk Mutect2 -R reference.fasta -I normal1.bam -O normal1.vcf.gz
gatk Mutect2 -R reference.fasta -I normal2.bam -O normal2.vcf.gz
Etc
```

- Step 2: 根据刚获得的各个样本的变异call，创建一个GenomicsDB，以提高规模效率：

```sh
gatk GenomicsDBImport -R reference.fasta -L intervals.interval_list \
  --genomicsdb-workspace-path pon_db \
  --max-mnp-distance 0 \
  -V normal1.vcf.gz \
  -V normal2.vcf.gz \
  -V normal3.vcf.gz
```

- Step 3: 获得PoN：

```sh
gatk CreateSomaticPanelOfNormals -R reference.fasta \
  --germline-resource af-only-gnomad.vcf.gz \
  -V gendb://pon_db \
  -O pon.vcf.gz
```



## Estimate cross-sample contamination using GetPileupSummaries and CalculateContamination

**[GetPileupSummaries](https://software.broadinstitute.org/gatk/documentation/tooldocs/current/org_broadinstitute_hellbender_tools_walkers_contamination_GetPileupSummaries.php)**: Summarizes counts of reads that support reference, alternate and other alleles for given sites. Results can be used with CalculateContamination.

**[CalculateContamination](https://software.broadinstitute.org/gatk/documentation/tooldocs/4.0.3.0/org_broadinstitute_hellbender_tools_walkers_contamination_CalculateContamination.php)**: 计算来自于交叉样本的read的份数，结果用于下游的FilterMutectCalls分析。

- Step 1: 

```sh
# 例子1
gatk GetPileupSummaries \
   -I tumor.bam \
   -V common_biallelic.vcf.gz \
   -L common_biallelic.vcf.gz \
   -O pileups.table
# 例子2
gatk GetPileupSummaries \
   -I normal.bam \
   -V common_biallelic.vcf.gz \
   -L common_biallelic.vcf.gz \
   -O pileups.table
# 例子3 -V和-L参数不一定相同
gatk GetPileupSummaries \
   -I normal.bam \
   -V gnomad.vcf.gz \
   -L common_snps.interval_list \
   -O pileups.table
```

`-I`：即`--input`，接受BAM/SAM/CRAM等含reads的文件

`-V`：即`--variant`，接受含有变异和等位基因频率的VCF文件

`-L`：即`--intervals`，指定在基因组特定区间进行分析



- Step 2：用CalculateContamination评估污染情况：

```sh
# Tumor-only mode
gatk CalculateContamination \
   -I pileups.table \
   -O contamination.table

# Matched normal mode
gatk CalculateContamination \
   -I tumor-pileups.table \
   -matched normal-pileups.table \
   -O contamination.table
```

`-matched`：即`--matched-normal`，现版本即使不提供也能运作良好。

`-segment`：即`--tumor-segmentation`，输出一个表格，The output table containing segmentation of the tumor by minor allele fraction。

## Filter for confident somatic calls using FilterMutectCalls

[FilterMutectCalls](https://software.broadinstitute.org/gatk/documentation/tooldocs/4.0.6.0/org_broadinstitute_hellbender_tools_walkers_mutect_FilterMutectCalls.php) example：

```sh
gatk FilterMutectCalls \
   -R ref.fasta \
   -V somatic.vcf.gz \
   -contamination-table contamination.table \
   -O filtered.vcf.gz
```



## (Optional) Estimate artifacts with CollectSequencingArtifactMetrics and filter them with FilterByOrientationBias

FilterByOrientationBias可以过滤如

OxoG： 8-Oxoguanine，G被氧化成8-oxoguanine，导致G-T的颠换，以及

FFPE：formalin-fixed paraffin-embedded，C在甲醛作用下脱氨，导致C-T的转换

等人工引起的变异。

- Step 1: 在运行Mutect2时，传入一个实参`--f1r2-tar-gz`：

```sh
gatk Mutect2 -R ref.fasta \
   -L intervals.interval_list \
   -I tumor.bam \
   -germline-resource af-only-gnomad.vcf \
   -pon panel_of_normals.vcf   \
   --f1r2-tar-gz f1r2.tar.gz \
   -O unfiltered.vcf
```

 生成的f1r2.tar.gz可被用于学习orientation bias model。

- Step 2: 将f1r2.tar.gz传给LearnReadOrientationModel：

```sh
gatk LearnReadOrientationModel -I f1r2.tar.gz -O read-orientation-model.tar.gz
```

- Step 3: 评估污染情况，如上述。
- Step 4: 将学习好的read orientation model作为实参`--ob-priors`传入FilterMutectCalls：

```sh
gatk FilterMutectCalls -R ref.fasta -V unfiltered.vcf \
   [--tumor-segmentation segments.table] \
   [--contamination-table contamination.table] \
   --ob-priors read-orientation-model.tar.gz \
   -O filtered.vcf
```



## Pratice

所有材料均下载自ftp.broadinstitute.org，目录结构如下：

```bash
somatic/
├── inputs
│   ├── 4_NA19771.vcf.gz
│   ├── 4_NA19771.vcf.gz.tbi
│   ├── 5_HG02759.vcf.gz
│   ├── 5_HG02759.vcf.gz.tbi
│   ├── af-only-gnomad.hg38.vcf.gz
│   ├── af-only-gnomad.hg38.vcf.gz.tbi
│   ├── chr17plus.interval_list
│   ├── chr17_small_exac_common_3_grch38.vcf.gz
│   ├── chr17_small_exac_common_3_grch38.vcf.gz.tbi
│   ├── HG00190.bai
│   ├── HG00190.bam
│   ├── normal.bai
│   ├── normal.bam
│   ├── tumor.bai
│   └── tumor.bam
├── oustputs
├── pondb
└── references
    ├── Homo_sapiens_assembly38.dict
    ├── Homo_sapiens_assembly38.fasta
    └── Homo_sapiens_assembly38.fasta.fai
```

#### 

```sh
# 用样本HG00190、NA19441、HG02759构建PoN
# 以HG00190为例，在Mutect2的tumor-only模式下生成3_HG00190.vcf.gz
time gatk Mutect2 \
-R references/Homo_sapiens_assembly38.fasta \
-I inputs/HG00190.bam \
-L inputs/chr17plus.interval_list \
--disable-read-filter MateOnSameContigOrNoMappedMateReadFilter \
-O inputs/3_HG00190.vcf.gz

# 注意生成的pon_db需要放置在不存在的或空的文件夹中
time gatk GenomicsDBImport \
  -R references/Homo_sapiens_assembly38.fasta \
  -L inputs/chr17plus.interval_list \
  -V inpust/3_HG00190.vcf.gz \
  -V inputs/4_NA19771.vcf.gz \
  -V inputs/5_HG02759.vcf.gz \
  --genomicsdb-workspace-path ~/somatic/pon_db/
  
# 创建pon，注意-V参数的格式
time gatk CreateSomaticPanelOfNormals \
  -R references/Homo_sapiens_assembly38.fasta \
  --germline-resource inputs/af-only-gnomad.hg38.vcf.gz \
  -V gendb://pon_db \
  -O outputs/pon.vcf.gz
  
# 用Mutect2做变异calling,
time gatk Mutect2 \
   -R references/Homo_sapiens_assembly38.fasta \
   -L inputs/chr17plus.interval_list \
   -I inputs/tumor.bam \
   -germline-resource inputs/af-only-gnomad.hg38.vcf.gz \
   -pon outputs/pon.vcf.gz   \
   --f1r2-tar-gz outputs/f1r2.tar.gz \
   -O outputs/unfiltered.vcf

# 将f1r2.tar.gz交给LearnReadOrientationModel学习
time gatk LearnReadOrientationModel \
   -I outputs/f1r2.tar.gz \
   -O outputs/read-orientation-model.tar.gz
   
# GetPileupSummaries
time gatk GetPileupSummaries \
    -I inputs/tumor.bam \
    -V inputs/chr17_small_exac_common_3_grch38.vcf.gz \
    -L inputs/chr17_small_exac_common_3_grch38.vcf.gz \
    -O outputs/getpileupsummaries.table

# 计算污染
time gatk CalculateContamination \
    -I outputs/getpileupsummaries.table \
    -tumor-segmentation outputs/segments.table \
    -O outputs/calculatecontamination.table

# 获得结果
time gatk FilterMutectCalls \
   -R references/Homo_sapiens_assembly38.fasta \
   -V outputs/unfiltered.vcf \
   --tumor-segmentation outputs/segments.table \
   -contamination-table outputs/calculatecontamination.table \
   --ob-priors outputs/read-orientation-model.tar.gz \
   -O outputs/somatic_filtered.vcf.gz
```

