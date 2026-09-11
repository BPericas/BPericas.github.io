---
layout: post
title: "Analysing Mismatch Negativity EEG data recorded with CURRY8"
description: Blog post explaining an MMN analysis pipeline created using MATLAB. The EEG data was collected using a Neuroscan 32-electrode system and CURRY8 software for EEG data recording. The analysis pipeline includes preprocessing, ERP extraction, and cluster-based permutation testing (FieldTrip). 
tags: [EEG data analysis, CURRY8, MATLAB, Mismatch response analysis]
---
This blog post aims to serve two purposes. First, it provides a more detailed guide to using the scripts in the mmn-gamified-phonetic-training repository to analyse mismatch negativity (MMN) responses. Second, it explains the rationale behind each preprocessing and analysis step included in the pipeline. As such, it may be useful for researchers who are transitioning to CURRY for EEG data collection or for those who are new to working with MMN responses.

When I joined the Lisbon Baby Lab for my first postdoctoral position, I started working with Neuroscan EEG systems and CURRY software for EEG data acquisition. I was responsible for the EEG component of a gamified phonetic training study involving Portuguese learners of English and was particularly interested in investigating how gamified phonetic training affected learners' MMN responses. During my PhD, I had collected and analysed a considerable amount of EEG data using Biosemi systems and MATLAB, so it made sense to continue using MATLAB for analysing the CURRY8 data. However, I quickly realised that there was not much guidance available online for processing CURRY8 EEG recordings in MATLAB, perhaps because CURRY comes with its own analysis software. Having now spent some time working with CURRY data and developing this analysis pipeline, I thought it would be useful to document the process in the form of this blog post in case it helps other researchers facing the same situation. 

**Preparing to run the pipeline**

Before running any of the scripts in the mmn-gamified-phonetic-training repository, there are a few software packages and toolboxes that need to be installed. The pipeline was developed and tested using MATLAB R2024b together with EEGLAB and FieldTrip, so I would recommend installing the same versions where possible. EEGLAB can be downloaded free of charge from the Swartz Centre for Computational Neuroscience website and should be added to your MATLAB path before running any analyses. The pipeline also relies on the FieldTrip toolbox, which is available from the FieldTrip website and provides several functions that are used throughout the preprocessing and ERP analysis stages. Moreover, EEGLAB has a CURRY import plugin which you can find on mattpontifex's GitHub. Download the loadcurry plugin and save it in your EEGLAB plugins folder. 

In addition to EEGLAB and FieldTrip, the artefact-removal stage requires the GEDAI (Generalised Eigenvalue De-Artifacting Instrument) toolbox, which is available from the GEDAI GitHub repository and can be used as an EEGLAB plugin. The pipeline also uses NoiseTools, a MATLAB toolbox for denoising and analysing electrophysiological data, which can be downloaded from Alain de Cheveigné's website. Finally, the ICLabel EEGLAB plugin is also required for automatic eye-blink component classification and can be installed easily through the EEGLAB extension manager.

It also important to note that the data was recorded using a NeuroScan 32-channel QuickCap with 30 EEG channels and 2 EOG channels. 

**List of links:**

MATLAB R2024b: https://www.mathworks.com/products/matlab.html

EEGLAB: https://eeglab.org

FieldTrip: https://www.fieldtriptoolbox.org/download/

GEDAI: https://github.com/NeuroEngUAB/GEDAI

NoiseTools: http://audition.ens.fr/adc/NoiseTools/

CURRY import plugin for EEGLAB (Matthew Pontifex): https://github.com/mattpontifex/loadcurry

ICLabel plugin: Available through the EEGLAB Extension Manager or https://github.com/sccn/ICLabel

**Preprocessing the EEG data**

The first step of the pipeline applies a 0.1 Hz high-pass filter to each recording block separately, rather than filtering the entire recording in one go. This approach helps to avoid edge artefacts at block boundaries. Before any channels are removed, the script saves the VEOG signal so that it can later be used to identify eye-blink related components. The VEOG and trigger channels are then removed, standard 10-20 electrode locations are assigned, and channel labels are converted from the NeuroScan naming convention (e.g., FZ, FCZ) to the mixed-case format used by GEDAI, the data-cleaning method I found most appropriate for my data, and FieldTrip (e.g., Fz, FCz). Finally, the processed data are saved as EEGLAB .set files together with MATLAB snapshots of the processed EEG and the separately stored VEOG signal.  

The second step of the pipeline involves artefact and rereferencing. It takes the filtered files and first applies GEDAI to reduce non-neural noise in the data. I chose GEDAI as the artifact removal method because it provides a largely automated approach to artefact removal and can be applied consistently across participants. This reduces the amount of manual intervention required during preprocessing and helps make the analysis pipeline more reproducible. After GEDAI cleaning, ICA is run on the EEG data, and ICLabel (from EEGLAB) is used to identify eye-blink related components. Importantly, components are only removed when they are classified as eye activity with high confidence and show the expected frontopolar scalp distribution. Although the VEOG signal saved in the previous step is loaded by the script, it is not used to drive artefact rejection. Instead, it is used as a post-ICA quality check by calculating residual correlations with the cleaned data. Finally, the EEG is rereferenced using the robust rereferencing method implemented in NoiseTools and saved for the subsequent stages of the MMN analysis pipeline. As a precaution, script number three in the pipeline generates ERP plots to visually inspect the cleaned data and verify that the expected auditory ERPs are still present. These plots act as sanity checks to confirm that the cleaning procedure has not been too aggressive.

Once the data have been cleaned, the next step is to extract the ERPs and compute the MMN response. The third script loads the artefact-cleaned EEG files, applies a 30 Hz low-pass filter, and epochs the data around standard and deviant stimuli using a 200 ms pre-stimulus baseline. After baseline correction, subject-level ERPs are calculated by averaging across trials, and MMN difference waves are computed by subtracting the standard ERP from the deviant ERP. The script then calculates grand-average ERPs and MMN responses across participants and generates plots showing the group-level responses. To accommodate occasional bad channels that may have been removed during preprocessing, grand averages are computed using only the channels that are available for all participants. As the MMN is typically observed as a negative deflection at frontocentral electrodes, the resulting difference wave should show a negative peak approximately 100 to 250 ms after stimulus onset.

The fourth script separates the MMN responses according to phonetic contrast and testing session. In our study, participants completed pre- and post-training EEG sessions and were presented with two different phonetic contrasts. Because the experiment used a block design, each block contained stimuli from only one phonetic contrast. The script uses a CSV file that records which contrast was presented in each block for each participant and uses this information to assign epochs to the correct contrast. Subject-level MMN difference waves (DEV − STD) are then calculated separately for each contrast at pre-test and post-test.

Splitting the data in this way allowed us to examine whether the effects of training differed across the two phonetic contrasts and whether any changes in MMN responses were observed after training. The resulting MMN waveforms are saved separately for each participant, contrast, and testing session, making them ready for the statistical analyses carried out in the next step of the pipeline. In addition, the script computes grand-average MMN waveforms for each contrast and time point and generates a set of figures comparing pre- and post-training responses. These plots provide another useful opportunity to visually inspect the data and check whether the expected MMN responses are present before moving on to the cluster-based permutation analyses.

**Running the stats**

The fifth script carries out a cluster-based permutation analysis to compare MMN responses across conditions. Rather than testing every time point individually, which would create a large multiple-comparisons problem, the analysis identifies clusters of neighbouring time points that show a consistent effect and then evaluates how likely it is that those clusters could have occurred by chance. This approach is widely used in EEG research because it takes into account the temporal dependence of ERP data while controlling the family-wise error rate.

One aspect of this analysis that I particularly liked was that I did not need to specify in advance which electrodes or time windows should contain the effect. Instead, the cluster-based permutation procedure searched across the data and identified clusters wherever reliable differences emerged. This reduces the risk of biasing the analysis towards a particular electrode site or latency range based on prior expectations. Given that training-related effects can sometimes differ from the canonical MMN pattern, allowing the analysis to identify significant clusters in a data-driven way seemed like a sensible approach.

The main output of this script is a MATLAB structure called stat, which contains information about the clusters identified by the analysis. When inspecting the results, I would recommend looking first at the posclusters and negclusters fields, which contain information about positive and negative clusters respectively. You can then examine the corresponding cluster labels to determine the time periods over which each cluster extends. This is useful because it allows you to see not only whether an effect was present, but also when it occurred. For MMN studies, it can be helpful to compare these time windows with the expected MMN latency range, typically around 100 to 250 ms after stimulus onset.

It is also important to inspect the prob field associated with each cluster. These probability values indicate whether a cluster is statistically significant under the permutation distribution generated by the analysis. Clusters with probability values below your chosen significance threshold, typically p < .05, can be interpreted as significant, whereas clusters with larger probability values should be treated cautiously. Together, the cluster information, probability values, and accompanying figures provide a useful overview of both the timing and strength of any training-related changes in MMN responses.

The sixth and final script uses the same cluster-based permutation approach described in the previous section, but this time the comparison is between pre-test and post-test MMN responses for each phonetic contrast. As in script 5, I did not specify particular electrodes or time windows of interest in advance. Instead, the analysis was allowed to identify significant clusters across the channel × time space in a data-driven way, reducing the potential for bias and allowing training-related effects to emerge directly from the data.

The main difference from the previous analysis is the question being asked. Rather than comparing MMN responses between phonetic contrasts, this script tests whether MMN responses changed following training. The analysis compares post-test and pre-test MMN waveforms using a dependent-samples t-test, as the comparison is within the same participants across two time points: pre-test and post-test. This analysis included only participants who contributed data to both pre- and post-test sessions. 

The main output is a file called ClusterPerm_PreVsPost_FT.mat, which contains a FieldTrip stat structure for each phonetic contrast. As in script 5, I would recommend looking at the positive and negative clusters identified by the analysis and inspecting their associated probability values (prob) to determine whether they are statistically significant. One important detail to keep in mind is that the comparison is calculated as Post − Pre. Because the MMN is a negative-going ERP component, a larger MMN after training will produce a negative cluster. In other words, significant negative clusters indicate that the MMN became more negative following training, whereas significant positive clusters indicate that the MMN became less negative at post-test.

As with the previous step, it is worth looking beyond the significance values alone and considering when and where any clusters occur. Comparing the significant clusters with the ERP waveforms and topographic plots can help determine whether the observed effects are consistent with the expected MMN response and can provide a more complete picture of how training influenced neural discrimination of the phonetic contrasts.

**A few final remarks**

I initially developed these scripts because I needed a practical way of analysing CURRY EEG data in MATLAB and found that there was relatively little information available online about how to do so. Over time, the analysis pipeline evolved into a set of scripts that could take the data from the raw recordings all the way through to the final statistical analyses while keeping the process as automated and reproducible as possible. Hopefully, this blog post has helped explain not only how to run the scripts, but also the reasoning behind the different preprocessing and analysis decisions that were made along the way.

Of course, no analysis pipeline is suitable for every study, so always consider whether particular preprocessing steps, parameters, or statistical approaches are appropriate for your own data and research questions. Nevertheless, if you are working with MMN data, are new to CURRY recordings, or simply want a starting point for building your own EEG analysis workflow, I hope that you find these scripts useful. The code is freely available on GitHub, and I encourage anyone using or adapting it to inspect the outputs at each stage of the pipeline rather than treating the analysis as a completely automated process. A few minutes spent checking waveforms, topographies, and statistical outputs can often save a lot of time later on.

Finally, if you spot any bugs, have suggestions for improving the pipeline, or adapt the scripts for your own research, I would be very interested to hear about it. One of the advantages of sharing analysis code openly is that it allows researchers to learn from one another and continuously improve the tools that we use to analyse EEG data.






