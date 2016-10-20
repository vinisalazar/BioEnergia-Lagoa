<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Workflow análise CBO</title>
<link rel="stylesheet" href="https://stackedit.io/res-min/themes/base.css" />
<script type="text/javascript" src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS_HTML"></script>
</head>
<body><div class="container"><h3 id="protocolo-de-análise-comunidade-microbiana-da-lagoa-da-conceição">Protocolo de análise - Comunidade microbiana da Lagoa da Conceição</h3>

<p>Esse protocolo tem a intenção de garantir a transparência e reproducibilidade da análise de dados realizada pelo Grupo de Oceanografia Microbiana - GOM UFSC para o projeto <strong>“Determinação de diversidade filogenética em zonas mortas (Lagoa da Conceição - Florianópolis / SC) com ênfase na identificação de bactérias produtoras de hidrogênio”</strong>.</p>

<p>Essa análise foi conduzida utilizando as ferramentas informáticas <strong>PEAR</strong> (<em>v0.9.10</em>), <strong>QIIME</strong> (<em>v1.9.1</em>), <strong>USEARCH</strong> (<em>v7.0</em>), a base de dados <strong>SILVA</strong> (<em>119</em>), baseada no protocolo desenvolvido pelo <a href="http://www.brmicrobiome.org/#!16s-profiling-pipeline-illumina/czxl">Brazilian Microbiome Project</a> <strong>(BMP)</strong>. Recomendamos consultar essa página do BMP para saber mais sobre a análise. Você pode encontrar as citações completas desses softwares nas referências bibliográficas da publicação a qual esse documento está relacionado. </p>

<p>Todos os comandos aqui são realizados no Terminal (bash).</p>

<p>Passo 1 - Mergindo os reads com <strong>PEAR</strong></p>

<p><code>$ pear -q 24 -n 350 -t 150 -u 0.02 -v 100 -j 8 -o output/amostra_mergida1 -f $PWD/amostra_forward1 -r $PWD/amostra_reverse1</code></p>

<p>Onde:  <br>
-q = quality_threshold <br>
-n = min_read_length <br>
-t = min_trim_length <br>
-u = max_uncalled_base <br>
-v = min_overlap <br>
-j = threads <br>
-o = output directory <br>
-f = forward_read <br>
-r = reverse_read</p>

<p>Fazemos isso para cada amostra.</p>

<p>Passo 2 - Processando as amostras com <strong>QIIME</strong></p>

<p><code>$ multiple_split_libraries_fastq.py -i $PWD/amostras -o slout -p ../split_par.txt</code></p>

<p>Conteúdo do arquivo split_par.txt: <br>
split_libraries_fastq:phred_quality_threshold   24 <br>
split_libraries_fastq:barcode_type  not_barcoded <br>
split_libraries_fastq:sequence_max_n    100 <br>
split_libraries_fastq:store_demultiplexed_fastq True</p>

<p>Passo 3 - Quality filtering com <strong>USEARCH 7</strong></p>

<p><code>$ usearch7 -fastq_filter slout/seqs.fastq -fastq_maxee 0.5 -fastq_trunclen 350 -fastaout reads350ee50.fa</code></p>

<p>Passo 4 - Conversão do QIIME para formato UPARSE</p>

<p><code>$ perl $PWD/bmp-Qiime2Uparse.pl -i reads350ee50.fa -o reads_uparse.fa</code></p>

<p>Passo 5 - <strong>USEARCH</strong>: derreplicação,  remoção de singletons, <em>de novo</em> clustering, remoção de quimeras</p>

<p><code>$ usearch7 -derep_fulllength reads_uparse.fa -output derep.fa -sizeout</code></p>

<p><code>$ usearch7 -sortbysize derep.fa -output sorted.fa -minsize 2</code></p>

<p><code>$ usearch7 -cluster_otus sorted.fa -otus otus1.fa</code></p>

<p><code>$ usearch7 -uchime_ref otus1.fa -db $PWD/gold.fa -strand plus -nonchimeras otus2.fa</code></p>

<p>Passo 6 - Formatação sugerida pelo BMP</p>

<p><code>$ fasta_formatter -i otus2.fa -o formated_otus2.fa</code> <br>
<code>$ perl $PWD/bmp-otuName.pl -i formated_otus2.fa -o otus.fa</code></p>

<p>Passo 7 - Mapeando os reads de volta a database de OTUs</p>

<p><code>$ usearch7 -usearch_global reads_uparse.fa -db otus.fa -strand plus -id 0.97 -uc map.uc</code></p>

<p>Passo 8 - atribuição de taxonomia no <strong>QIIME</strong> usando a base de dados <strong>Silva 119</strong>, e subsequente alinhamento, filtragem de alinhamento e construção de árvore.</p>

<p><code>$ assign_taxonomy.py -i otus.fa -o tax_out_silva -r $PWD/Silva119_release/rep_set/97/Silva_119_rep_set97.fna -t $PWD/Silva119_release/taxonomy/97/taxonomy_97_7_levels.txt</code></p>

<p><code>$ align_seqs.py -i otus.fa -o rep_set_align -t $PWD/rep_set_align/Silva_119_rep_set97_aligned_16S_only.fna</code></p>

<p><code>$ filter_alignment.py -i rep_set_align/otus_aligned.fasta -o filtered_alignment</code></p>

<p><code>$ make_phylogeny.py -i filtered_alignment/otus_aligned_pfiltered.fasta -o rep_set.tre</code></p>

<p>Passo 9 - Formatação sugerida pelo BMP</p>

<p><code>$ python $PWD/bmp-map2qiime.py map.uc &gt; otu_table.txt</code></p>

<p>Passo 10 - Construção da tabela .biom</p>

<p><code>$ make_otu_table.py -i otu_table.txt -t tax_out_silva/otus_tax_assignments.txt -o otu_table.biom</code></p>

<p>Olhando nossa tabela BIOM, podemos determinar o valor de subamostragem que foi usado para as análises de diversidade:</p>

<p><code>$ biom summarize-table -i otu_table.biom</code></p>

<p><code>Num samples: 16 <br>
Num observations: 1546 <br>
Total count: 219709 <br>
Table density (fraction of non-zero values): 0.707</code></p>

<p><code>Counts/sample summary: <br>
 Min: 7488.0 <br>
 Max: 22498.0 <br>
 Median: 12012.500 <br>
 Mean: 13731.812 <br>
 Std. dev.: 4670.947 <br>
 Sample Metadata Categories: None provided <br>
 Observation Metadata Categories: taxonomy</code></p>

<p><code>Counts/sample detail: <br>
82.Nov.a.assembled.fastq: 7488.0 <br>
33.Dez.a.assembled.fastq: 7546.0 <br>
82.Mar.b.assembled.fastq: 9503.0 <br>
82.Nov.b.assembled.fastq: 10135.0 <br>
82.Dez.a.assembled.fastq: 10780.0 <br>
33.Nov.a.assembled.fastq: 11100.0 <br>
82.Dez.b.assembled.fastq: 11660.0 <br>
33.Jan.b.assembled.fastq: 11749.0 <br>
82.Jan.a.assembled.fastq: 12276.0 <br>
33.Mar.b.assembled.fastq: 12469.0 <br>
33.Nov.b.assembled.fastq: 15665.0 <br>
82.Jan.b.assembled.fastq: 16197.0 <br>
33.Jan.a.assembled.fastq: 19868.0 <br>
33.Dez.b.assembled.fastq: 19874.0 <br>
33.Mar.a.assembled.fastq: 20901.0 <br>
82.Mar.a.assembled.fastq: 22498.0</code></p>

<p>Passo 11 - Análise de diversidade: parâmetros <em>default</em> do <strong>QIIME</strong>, exceto pelo valor -e (subamostragem) obtido através da análise de nossa tabela BIOM. Para essa análise, o valor utilizado foi -e 7488 (valor da amostra com menos observações):</p>

<p><code>$ core_diversity_analyses.py -i otu_table.biom -m map.txt -t rep_set.tre -e 7488 -o core_output</code></p>

<p>Nesse projeto, qualquer resultado apresentado como “gerado pelo QIIME” é um output do comando acima.</p>

<p>Correspondência/dúvidas: <a href="https://github.com/vinisalazar">GitHub/vinisalazar</a> - viniws@gmail.com</p>

<blockquote>
  <p>Written with <a href="https://stackedit.io/">StackEdit</a>.</p>
</blockquote></div></body>
</html>