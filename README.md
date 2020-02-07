# inGAP-CDG


Manual of inGAP-CDG v1.2


Description: 
inGAP-CDG is a novel computational method for effective construction of full-length and non-redundant CDSs from unassembled transcriptomes. inGAP-CDG achieves this by combining a newly developed codon-based de bruijn graph to simplify the assembly process and a machine learning based approach to filter false positives. Compared with other methods, inGAP-CDG exhibits significantly increased predicted CDS length and robustness to sequencing errors and varied read length. It contains two algorithms: inGAP-CDG_readToCDS and inGAP-CDG_transcriptToCDS. 

inGAP-CDG_readToCDS is designed to predict genes from unassembled RNA-seq reads.
inGAP-CDG_TranscriptToCDS is designed to predict genes from assembled transcripts.

Contents

1.  Installation
2.  inGAP-CDG USAGE: command options        
        (1) inGAP-CDG_readToCDS
        (2) inGAP-CDG_transcriptToCDS
3.  inGAP-CDG RUN: common commands and examples
4.  inGAP-CDG output
5.  FAQ

1.  Installation
============================================================================================================

 # Linux 64-bit system

    Requirements:
    GCC: 4.1.2 or later
    OpenMP (http://openmp.org/)

    $ tar xzf inGAP-CDG_linux_x64_v1.2.tar.gz
    $ cd inGAP-CDG_linux_x64_v1.2

    $ make 
    optionally, $ sh install.sh 

 # Mac OSX 64-bit system

    Requirements:
    GCC: 4.1.2 or later

    $ tar xzf inGAP-CDG_macosx_x64_v1.2.tar.gz
    $ cd inGAP-CDG_macosx_x64_v1.2

    $ make
    optionally, $ sh install.sh


2.  inGAP-CDG USAGE: command options
============================================================================================================
     (1) inGAP-CDG_readToCDS
         
          ./inGAP-CDG_readToCDS  [options]
         Options: 
          -i, --input_file             Please enter your input filename (in fasta format). [required]
          -o, --output_dir             Please enter your output directory filename. If not exists, the program will create it.  
          -n, --threads_num            The thread number supported by openmp. [default: 1]

          -L, --train_seq_len          The minimal length of CDSs in positive data set for SVM. [default: 1000] 
          -l, --potential_ORFs_cutoff  The potential ORFs that are larger than --potential_ORFs_cutoff*read_length will be kept. [default: 0.8] 
          -d, --svm_dev                The SVM classification vector value to filter false positive ORFs. [default: 0] [-0.1, 0.1]

          -k, --kmer_length            The kmer size (a triple number) used to construct codon-based de Bruijn graph. [default: 27] 
          -p, —-subgraph_size          The minimal number of subgraph size used to traverse. [default: 300]
          -t, --tips_length            The cutoff length of tips to be trimmed in de Bruijn graph. [default: 2*kmer_length]

          -h, --help                   Display the help information for options.

     (2) inGAP-CDG_transcriptToCDS
         
          ./inGAP-CDG_transcriptToCDS  [options]
         Options: 
          -i, --input_file             Please enter your input filename (in fasta format). [required]
          -o, --output_dir             Please enter your output directory filename. If not exists, the program will create it. 
          -n, --threads_num            The thread number supported by openmp. [default: 1]

          -L, --train_seq_len          The minimal length of CDSs in positive data set for SVM. [default: 1500] 
          -l, --test_seq_len           The length of test sequence for SVM. [default: 100]
          -d, --svm_dev                The SVM classification vector value to filter false positive ORFs. [default: 0] [-0.1, 0.1]

          -k, --kmer_length            The kmer size (a triple number) used to construct codon-based de Bruijn graph. [default: 27] 
          -p, —-subgraph_size          The minimal number of subgraph size used to traverse. [default: 300]
          -t, --tips_length            The cutoff length of tips to be trimmed in de Bruijn graph. [default: 2*kmer_length]
          
          -h, --help                   Display the help information for options.

     
3.  inGAP-CDG RUN: common commands and examples 
===================================================================================================

    NOTE:This manual assumes that you have all the software needed installed on your system and can thus directly run them. If you have it not installed but only downloaded and extracted to a folder, please use the absolute path instead.

    (1) Download reads from NCBI SRA database
    
    Take a human RNA-seq data set with the accession number of SRR1045067 as an example. You can download this data set from SRA(Sequence Read Archive) database of NCBI(National Center for Biotechnology Information) using following command:
   
         wget ftp://ftp.ncbi.nlm.nih.gov/sra/sra-instant/reads/ByRun/sra/SRR/SRR104/SRR1045067/SRR1045067.sra
   
    (2) Using the SRA Toolkit (https://trace.ncbi.nlm.nih.gov/Traces/sra/sra.cgi?view=software) to convert the downloaded file of SRR1045067.sra into ‘fastq’ formats
   
        fastq-dump —-split-3 SRR1045067.sra 

    (3) Using Trimmomatic (http://www.usadellab.org/cms/index.php?page=trimmomatic) to trim low quality reads
  
        java -jar ./Trimmomatic-0.30/trimmomatic-0.30.jar PE -threads 20 -phred33 SRR1045067_1.fastq SRR1045067_2.fastq  SRR1045067_1.clean.fastq SRR1045067_1.unpaired.fastq SRR1045067_2.clean.fastq SRR1045067_2.unpaired.fastq  ILLUMINACLIP:/Trimmomatic-0.30/adapters/TruSeq3-PE.fa:2:30:10  LEADING:3 TRAILING:3 SLIDINGWINDOW:4:15 MINLEN:100
   
    (4) Merging reads 
    
      1) if the paired-end reads have overlaps, you can use FLASH to merge them into long reads (https://sourceforge.net/projects/flashpage/)

            flash -r 150 -f 250 -s 20 SRR1045067_1.clean.fastq SRR1045067_2.clean.fastq -o out
	
       FLASH will merge the paired-end reads into file ’out.extendedFrags.fastq’ and then, you can convert the .fq file to .fa file
          
          awk ‘{if(NR%4==1 || NR%4==2) print}’ out.extendedFrags.fastq | sed ’s/@/>/g’ > out.extendedFrags.fa
       
       2) if the paired-end reads have no overlaps, inGAP-CDG will treat them as single-end reads.
          
           cat SRR1045067_1.clean.fastq SRR1045067_2.clean.fastq | awk ‘{if(NR%4==1 || NR%4==2) print}’ | sed ’s/@/>/g’ > out.extendedFrags.fa
    
    (5) Running inGAP-CDG
    
       i. gene prediction based on reads

            inGAP-CDG_readToCDS -i out.extendedFrags.fa -o $your_output_dir [options]
        
        When finished, the resulting gene prediction file will be in the output/OutputCDSs folder.
         
        ii. gene prediction based on transcripts
         First, you need to assemble RNA-seq reads into transcripts using any transcriptome assembler. (e.g. Trinity, Soapdenovo-Trans or Oases). If you get the transcripts file named ‘transcripts.fas’, you will run inGAP-CDG using the following command:
          
           inGAP-CDG_transcriptToCDS -i transcripts.fas -o $your_output_dir [options]
          
        When finished, the resulting gene prediction file will be in the output/OutputCDSs folder.
     
    (6) Testing inGAP-CDG 

       You can test inGAP-CDG using the example file in the inGAP-CDG. The “example.fas” was sampled from a human RNA-seq data set with the accession number of SRR1045067. Specifically, paired-end reads were firstly merged by FLASH and then 500,000 merged sequences were randomly extracted to generate the sample data. Running inGAP-CDG can like as following command:
      
       inGAP-CDG_readToCDS -i example.fas -o $your_output_dir -n 8 -L 1000 [options]
       
       We have tested it successfully on Mac OS X EI Capitan (10.11) and Linux (Red Hat 6.3, Ubuntu 16.04, Ubuntu 14.04 LTS and Ubuntu 12.04 LTS) systems. 



    (7) Note 
       1) If you want to run inGAP-CDG in a Strict Mode (to increase CDS length and specificity but at the expense of sensitivity), you can simply increase the value of the ‘-p’ option from 300 [default] to 400 or 450. For example, inGAP-CDG_readToCDS -i example.fas -o $your_output_dir -n 8 -L 1000 -p 400 [options].

       2) The user can replace the blat binary optionally in the ‘tools’ directory of inGAP-CDG package using the latest version downloaded from owner’s page.




4.  FAQ
=============================================================================

For technical supports or report bugs, please send an email to penggongxin@biols.ac.cn.










