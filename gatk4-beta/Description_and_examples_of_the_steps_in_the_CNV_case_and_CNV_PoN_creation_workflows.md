## Description and examples of the steps in the CNV case and CNV PoN creation workflows

By LeeTL1220

<p>The CNV case and PoN workflows (description and examples) for earlier releases of GATK4.</p>

<h3>For a newer tutorial using GATK4's v1.0.0.0-alpha1.2.3 release (Version:0288cff-SNAPSHOT from September 2016), see <a rel="nofollow" href="http://gatkforums.broadinstitute.org/gatk/discussion/9143">Article#9143</a> and <a rel="nofollow" href="https://software.broadinstitute.org/gatk/download/workshops">this data bundle</a>. If you have a question on the Somatic_CNV_handson tutorial, please post it as a new question using <a rel="nofollow" href="http://gatkforums.broadinstitute.org/gatk/post/question/ask-the-team">this form</a>.</h3>

<hr></hr><h4>Requirements</h4>

<ol><li>Java 1.8</li>
<li>A functioning GATK4-protected jar (hellbender-protected.jar or gatk-protected.jar)</li>
<li>HDF5 1.8.13</li>
<li>The location of the HDF5-Java JNI Libraries Release 2.9 (2.11 for Macs).<br>
Typical locations:<br>
Ubuntu: <code class="code codeInline" spellcheck="false">/usr/lib/jni/</code><br>
Mac: <code class="code codeInline" spellcheck="false">/Applications/HDFView.app/Contents/Resources/lib/</code><br>
Broad internal servers: <code class="code codeInline" spellcheck="false">/broad/software/free/Linux/redhat_6_x86_64/pkgs/hdfview_2.9/HDFView/lib/linux/</code></li>
<li>Reference genome (fasta files) with fai and dict files.  This can be downloaded as part of the GATK resource bundle: <a href="http://www.broadinstitute.org/gatk/guide/article?id=1213" rel="nofollow">http://www.broadinstitute.org/gatk/guide/article?id=1213</a></li>
<li>PoN file (when running case samples only).  This file should be created using the Create PoN workflow (see below).<br><a name="SampleName" id="SampleName"></a><a></a></li>
<li>Target BED file that was used to create the PoN file.  Format details can be found <a rel="nofollow" href="http://genome.ucsc.edu/FAQ/FAQformat.html#format1">here</a> .  <strong>NOTE:</strong>  For the CNV tools, you will need a fourth column for target name, which must be unique across rows.</li>
</ol><pre class="code codeBlock" spellcheck="false">1       12200   12275   target1
1       13505   13600   target2
1       31000   31500   target3
1       35138   35174   target4
....snip....
</pre>

<p><em>Before running the workflows, we recommend padding the target file by 250 bases with the <code class="code codeInline" spellcheck="false">PadTargets</code> tool</em>.  Example:  <code class="code codeInline" spellcheck="false">java -jar gatk-protected.jar PadTargets --targets initial_target_file.bed --output initial_target_file.padded.bed --padding 250</code><br>
This allows some off-target reads to be factored into the copy ratio estimates.  Our internal evaluations have shown that this improves results.<br>
If you are using the premade Queue scripts (see below), you can specify the padding there and the workflow will generate the padded targets automatically (i.e. there is no reason to run PadTargets explicitly if you are using the premade Queue scripts).</p>

<h4>Case sample workflow</h4>

<p>This workflow requires a PoN file generated by the Create PoN workflow.</p>

<p>If you do not have a PoN, please skip to the <a rel="nofollow" href="#create-pon-workflow">Create PoN workflow</a>, below ....</p>

<h5>Overview of steps</h5>

<ul><li>Step 0. (recommended) Pad Targets (see example above)</li>
<li>Step 1. Collect proportional coverage</li>
<li>Step 2. Create coverage profile</li>
<li>Step 3. Segment coverage profile</li>
<li>Step 4. Plot coverage profile</li>
<li>Step 5. Call segments<br><a name="step-1-collect-proportional-coverage"></a></li>
</ul><h5>Step 1. Collect proportional coverage</h5>

<h6>Inputs</h6>

<ul><li>bam file</li>
<li>target bed file -- must be the same that was used for the PoN</li>
<li>reference_sequence (required by GATK) -- fasta file with b37 reference.</li>
</ul><h6>Outputs</h6>

<ul><li>Proportional coverage tsv file -- Mx5 matrix of proportional coverage, where M is the number of targets.  The fifth column will be named for the sample in the bam file (found in the bam file <code class="code codeInline" spellcheck="false">SM</code> tag).  If the file exists, it will be overwritten.</li>
</ul><pre class="code codeBlock" spellcheck="false">##fileFormat  = tsv
##commandLine = org.broadinstitute.hellbender.tools.exome.ExomeReadCounts  ...snip...
##title       = Read counts per target and sample
CONTIG  START   END     NAME    SAMPLE1
1       12200   12275   target1    1.150e-05
1       13505   13600   target2    1.500e-05
1       31000   31500   target3    7.000e-05
....snip....
</pre>

<h6>Invocation</h6>

<pre class="code codeBlock" spellcheck="false"> java -Xmx8g -jar &lt;path_to_hellbender_protected_jar&gt; CalculateTargetCoverage -I &lt;input_bam_file&gt; -O &lt;pcov_output_file_path&gt;  --targets &lt;target_BED&gt; -R &lt;ref_genome&gt; \ 
       -transform PCOV --targetInformationColumns FULL -groupBy SAMPLE -keepdups
</pre>

<h5>Step 2. Create coverage profile</h5>

<h6>Inputs</h6>

<ul><li>proportional coverage file from Step 1</li>
<li>target BED file -- must be the same that was used for the PoN</li>
<li>PoN file</li>
<li>directory containing the HDF5 JNI native libraries</li>
</ul><h6>Outputs</h6>

<ul><li>normalized coverage file (tsv) -- details each target with chromosome, start, end, and log copy ratio estimate</li>
</ul><pre class="code codeBlock" spellcheck="false">#fileFormat = tsv
#commandLine = ....snip....
#title = ....snip....
name    contig  start   stop    SAMPLE1
target1    1       12200   12275   -0.5958351605220968
target2    1       13505   13600   -0.2855054918109098
target3    1       31000   31500   -0.11450116047248263
....snip....
</pre>

<ul><li>pre-tangent-normalization coverage file (tsv) -- same as normalized coverage file (tsv) above, but copy ratio estimates are before the noise reduction step.  The file format is the same as the normalized coverage file (tsv).</li>
<li>fnt file (tsv) -- proportional coverage divided by the target factors contained in the PoN.  The file format is the same as the proportional coverage in step 1.</li>
<li>betaHats (tsv) -- used by developers and evaluators, typically, but output location must be specified.  These are the <br>
coefficients used in the projection of the case sample into the (reducued) PoN.  This will be a Mx1 matrix where M is the number of targets.</li>
</ul><h6>Invocation</h6>

<pre class="code codeBlock" spellcheck="false">java -Djava.library.path=&lt;hdf_jni_native_dir&gt; -Xmx8g -jar &lt;path_to_hellbender_protected_jar&gt; NormalizeSomaticReadCounts -I &lt;pcov_input_file_path&gt; -T &lt;target_BED&gt; -pon &lt;pon_file&gt; \
 -O &lt;output_target_cr_file&gt; -FNO &lt;output_target_fnt_file&gt; -BHO &lt;output_beta_hats_file&gt; -PTNO &lt;output_pre_tangent_normalization_cr_file&gt;
</pre>

<h5>Step 3. Segment coverage profile</h5>

<h6>Inputs</h6>

<ul><li>normalized coverage file (tsv) -- from step 2.</li>
<li>sample name</li>
</ul><h6>Outputs</h6>

<ul><li>seg file (tsv) -- segment file (tsv) detailing contig, start, end, and copy ratio (segment_mean) for each detected segment.  Note that this is a different format than python recapseg, since the segment mean no longer has log2 applied.</li>
</ul><pre class="code codeBlock" spellcheck="false">Sample  Chromosome      Start   End     Num_Probes      Segment_Mean
SAMPLE1        1       12200   70000   18       0.841235
SAMPLE1        1       300600  1630000 337     1.23232323
....snip....
</pre>

<h6>Invocation</h6>

<pre class="code codeBlock" spellcheck="false">java -Xmx8g -jar &lt;path_to_hellbender_protected_jar&gt;  PerformSegmentation  -S &lt;sample_name&gt; -T &lt;normalized_coverage_file&gt; -O &lt;output_seg_file&gt; -log
</pre>

<h5>Step 4. Plot coverage profile</h5>

<h6>Inputs</h6>

<ul><li>normalized coverage file (tsv) -- from step 2.</li>
<li>pre-normalized coverage file (tsv) -- from step 2.</li>
<li>segmented coverage file (seg) -- from step 3.</li>
<li>sample name, <a rel="nofollow" href="#SampleName">see above</a></li>
</ul><h6>Outputs</h6>

<ul><li>beforeAfterTangentLimPlot (png) -- Output before/after tangent normalization plot up to copy-ratio 4</li>
<li>beforeAfterTangentPlot (png) -- Output before/after tangent normalization plot</li>
<li>fullGenomePlot (png) -- Full genome plot after tangent normalization</li>
<li>preQc (txt) -- Median absolute differences of targets before normalization</li>
<li>postQc (txt) -- Median absolute differences of targets after normalization</li>
<li>dQc (txt) -- Difference in median absolute differences of targets before and after normalization</li>
</ul><h6>Invocation</h6>

<pre class="code codeBlock" spellcheck="false">java -Xmx8g -jar &lt;path_to_hellbender_protected_jar&gt;  PlotSegmentedCopyRatio  -S &lt;sample_name&gt; -T &lt;normalized_coverage_file&gt; -P &lt;pre_normalized_coverage_file&gt; -seg &lt;segmented_coverage_file&gt; -O &lt;output_seg_file&gt; -log
</pre>

<h5>Step 5. Call segments</h5>

<h6>Inputs</h6>

<ul><li>normalized coverage file (tsv) -- from step 2.</li>
<li>seg file (tsv) -- from step 3.</li>
<li>sample name</li>
</ul><h6>Outputs</h6>

<ul><li>called file (tsv) -- output is exactly the same as in seg file (step 3), except Segment_Call column is added.  Calls are either "+", "0", or "-" (no quotes).</li>
</ul><pre class="code codeBlock" spellcheck="false">Sample  Chromosome      Start   End     Num_Probes      Segment_Mean      Segment_Call
SAMPLE1        1       12200   70000   18       0.841235      -
SAMPLE1        1       300600  1630000 337     1.23232323     0 
....snip....
</pre>

<h6>Invocation</h6>

<pre class="code codeBlock" spellcheck="false">java -Xmx8g -jar &lt;path_to_hellbender_protected_jar&gt; CallSegments -T &lt;normalized_coverage_file&gt; -S &lt;seg_file&gt; -O &lt;output_called_seg_file&gt; -sample &lt;sample_name&gt; 
</pre>

<p><a name="create-pon-workflow"></a></p>

<h4>Create PoN workflow</h4>

<p>This workflow can take some time to run depending on how many samples are going into your PoN and the number of targets you are covering.  Basic time estimates are found in the <a rel="nofollow" href="#pon-overview-of-steps">Overview of Steps</a>.</p>

<h5>Additional requirements</h5>

<ul><li>Normal sample bam files to be used in the PoN.  The index files (.bai) must be local to all of the associated bam files.</li>
</ul><p><a name="pon-overview-of-steps"></a></p>

<h5>Overview of steps</h5>

<ul><li>Step 1. Collect proportional coverage.  (~20 minutes for mean 150x coverage and 150k targets, per sample)</li>
<li>Step 2. Combine proportional coverage files  (&lt; 5 minutes for 150k targets and 300 samples)</li>
<li>Step 3. Create the PoN file (~1.75 hours for 150k targets and 300 samples)</li>
</ul><p>All time estimates are using the internal Broad infrastructure.</p>

<h5>Step 1. Collect proportional coverage on each bam file</h5>

<p>This is exactly the same as the case sample workflow, except that this needs to be run once for each input bam file, each with a different output file name.  Otherwise, the inputs should be the same for each bam file.</p>

<p>Please see documentation <a rel="nofollow" href="#step-1-collect-proportional-coverage">above</a>.</p>

<p>IMPORTANT NOTE: You must create a list of the proportional coverage files (i.e. output files) that you create in this step.  One output file per line in a text file (see step 2)</p>

<h5>Step 2. Merge proportional coverage files</h5>

<p>This step merges the proportional coverage files into one large file with a separate column for each samples.</p>

<h6>Inputs</h6>

<ul><li>list of proportional coverage files generated (possibly manually) in step 1.  This is a text file.</li>
</ul><pre class="code codeBlock" spellcheck="false">/path/to/pcov_file1.txt
/path/to/pcov_file2.txt
/path/to/pcov_file3.txt
....snip....
</pre>

<h6>Outputs</h6>

<ul><li>merged tsv of proportional coverage</li>
</ul><pre class="code codeBlock" spellcheck="false">CONTIG  START   END     NAME    SAMPLE1    SAMPLE2 SAMPLE3 ....snip....
1       12191   12227   target1    8.835E-6  1.451E-5     1.221E-5    ....snip....
1       12596   12721   target2    1.602E-5  1.534E-5     1.318E-5   ....snip....
....snip....
</pre>

<h6>Invocation</h6>

<pre class="code codeBlock" spellcheck="false">java -Xmx8g -jar  &lt;path_to_hellbender_protected_jar&gt; CombineReadCounts --inputList &lt;text_file_list_of_proportional_coverage_files&gt; \
    -O &lt;output_merged_file&gt; -MOF 200 
</pre>

<h5>Step 3. Create the PoN file</h5>

<h6>Inputs</h6>

<ul><li>merged tsv of proportional coverage -- generated in step 2.</li>
</ul><h6>Outputs</h6>

<ul><li>PoN file -- HDF5 format.  This file can be used for running case samples sequenced with the same process.</li>
</ul><h6>Invocation</h6>

<pre class="code codeBlock" spellcheck="false">java -Xmx16g -Djava.library.path=&lt;hdf_jni_native_dir&gt; -jar &lt;path_to_hellbender_protected_jar&gt; CreatePanelOfNormals -I &lt;merged_pcov_file&gt; \
       -O &lt;output_pon_file_full_path&gt;
</pre>