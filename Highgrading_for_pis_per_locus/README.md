# No_of_pis_per_locus
Originally at: https://github.com/laninsky/Highgrading_for_pis_per_locus

You might need to filter your UCE data just to retain the most variable loci for *BEAST etc

### Getting the number of PIs per locus
For a given folder full of your UCE loci as individual nexus alignments (see fasta code below), the following R code will spit out a list of your loci, the number of parsimony informative sites (pis), and the length of the locus. 

```
#install phyloch and all its necessary packages (e.g. ape, colorspace, XML)
library(phyloch)
#Change to your working directory

# getting all the files
listoffiles <- list.files(pattern="*.nex*")
nooffiles <- length(listoffiles)

record <- c("locusname","pis","length")

for (j in 1:nooffiles) {
write.table((gsub("?","N",(readLines(listoffiles[j])),fixed=TRUE)),"list_of_pis_by_locus.txt",sep="",quote=FALSE,row.names=FALSE,col.names=FALSE)
tempfile <- read.nex("list_of_pis_by_locus.txt")
templength <- dim(tempfile)[2]
temppis <- pis(tempfile)
temp <- cbind(listoffiles[j],temppis,templength)
record <- rbind(record,temp)
}

write.table(record, "list_of_pis_by_locus.txt",quote=FALSE, row.names=FALSE,col.names=FALSE)

```

fasta code
```
#install phyloch and all its necessary packages (e.g. ape, colorspace, XML)
library(phyloch)
#Change to your working directory

# getting all the files
listoffiles <- list.files(pattern="*.fa*")
nooffiles <- length(listoffiles)

record <- c("locusname","pis","length")

for (j in 1:nooffiles) {
write.table((gsub("?","N",(readLines(listoffiles[j])),fixed=TRUE)),"list_of_pis_by_locus.txt",sep="",quote=FALSE,row.names=FALSE,col.names=FALSE)
tempfile <- read.fas("list_of_pis_by_locus.txt")
templength <- dim(tempfile)[2]
temppis <- pis(tempfile)
temp <- cbind(listoffiles[j],temppis,templength)
record <- rbind(record,temp)
}

write.table(record, "list_of_pis_by_locus.txt",quote=FALSE, row.names=FALSE,col.names=FALSE)

```

You can also use this list to summarize the number of pis in loci across different datasets using the code at: https://github.com/laninsky/comparing_lists

### Pruning gene trees corresponding to less variable loci from your total trees file
After I've run RAxML on my 'complete' dataset, I then use a modification of the file spat out above to prune the total gene trees file for all the loci, to only the more variable ones. To do this, you need to modify the "list_of_pis_by_locus.txt" to just the first column with the loci names, containing the loci you want to get rid of out of your file (and stripping any file suffixes e.g. 'nexus' from the names). Call this list "remove_list.txt". 

You then need to navigate to your complete_genetrees folder, and run the code at #6 at the following link in order to get a tree file which has the locus names given explicitly:
https://github.com/laninsky/Phase_hybrid_from_next_gen/tree/master/post-processing

After this, running the following code in R should constrain the trees in the "pitrees.tre" file to just those NOT on the remove list. It will also print out a file "whitelist.txt" which lists the retained loci. You are going to need this to filter the bootstrap replicates.

```
# remove_list.txt is a text file with each locus to remove on a new line (e.g. one column)
tocull <- as.matrix(read.csv("remove_list.txt",header=FALSE))

treefile <- as.matrix(read.table("ubertree.tre",sep=" ",header=FALSE))

output <- treefile[(!(treefile[,1] %in% tocull)),2]

whitelist <- treefile[(!(treefile[,1] %in% tocull)),1]

write.table(output, "pitrees.tre", sep="",quote=FALSE, row.names=FALSE,col.names=FALSE)

write.table(whitelist, "whitelist.txt", sep="",quote=FALSE, row.names=FALSE,col.names=FALSE)
```

I would suggest then moving the pitrees.tre and whitelist.txt files to a new working folder, and putting them in subfolders "complete" and "boots", respectively. Navigate to the boots directory where the whitelist.txt file is. Using this file, we are going to pull the 'whitelist' loci's bootstrapped trees from the complete dataset using bash, so we that don't have to do this step again. Set DEST as the folder that contains the bootstrapped loci e.g. "complete_bootstraps". This step will take a while, but not as much time as having to run RAxML on all your bootstrapped data from scratch! Change numboots to whatever the number of bootstraps you did originally was, minus one
```
DEST="/scratch/a499a400/gekko/complete_bootstraps"
noloci=`wc -l whitelist.txt | awk '{print $1}'`
numboots=499
bootplusone=$(expr $numboots + 1)
echo $bootplusone >> params
echo $noloci >> params

for i in `seq 1 $noloci`;
do locusname=`tail -n+$i whitelist.txt | head -n1`;
cat  $DEST/$locusname/RAxML_bootstrap.bootrep.$locusname >> combinedtrees
done;

wc -l combinedtrees | awk '{print $1}' >> params
```

There aren't exactly n=500 bootstrap reps per locus, because the bootstrapping process is taking place at sites within loci, and then across loci as well, so the following R code loops over the total number of trees we have concatenated together.

```
params <- read.table("params")
treefile <- as.matrix(read.table("combinedtrees"))

x <- 1
j <- 2

mastertemp <- NULL

for (i in 1:params[1,1]) {
temp <- NULL
while (is.null(temp) || dim(temp)[1] < params[2,1]) {
temp <- rbind(temp,treefile[x,1])
x <- x+500
if (x > params[3,1]) {
x <- j
j <- j + 1
}
}
varname <- paste("boot",(i-1),sep="")
write.table(temp,varname,sep="",quote=FALSE, row.names=FALSE,col.names=FALSE)
}
```

Using the boot0-boot499 files, you can then repeat your bootstrapping on these high-graded loci following along with the steps at: 
https://github.com/laninsky/UCE_processing_steps

### Pruning your nexus alignments of less variable loci
In addition, you might be interested in high-grading for more informative loci when doing a concatenated run of RAxML/exabayes on your complete/incomplete data. To do this, you can use the following code to move the nexus files for your 'less informative' loci (based on the remove_list.txt file you made) to another subfolder. To use this code, make sure remove_list.txt is in the folder with the nexus files that you want to highgrade for. It will transfer the less variable loci in your "remove_list.txt" to the "less_variable" folder.

```
mkdir less_variable
noloci=`wc -l remove_list.txt | awk '{print $1}'`

for i in `seq 1 $noloci`;
do locusname=`tail -n+$i remove_list.txt | head -n1`;
mv $locusname.nexus less_variable/
done;

mv remove_list.txt less_variable/
```

After doing this, you can carry on at Step8A of https://github.com/laninsky/UCE_processing_steps to generate your new RAxML files.

### Version history
v0.0 version used in gekko TBD.

### This pipeline wouldn't be possible without:

R: R Core Team. 2015. R: A language and environment for statistical computing. URL http://www.R-project.org/. R Foundation for Statistical Computing, Vienna, Austria. https://www.r-project.org/

Phyloch and its package dependencies: https://cran.r-project.org/web/packages/ips/citation.html
