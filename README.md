## WGCNA
## Download data to follow along: "MB.0.03.subsample.fn.shared", "OxygenMatrixMonterey.csv"


## Pre-WGCNA Data preparation

## Read in raw abundance values of OTUs
## Note: OTUs represent 16S rRNA sequences that were assessed with the universal primers 515F-Y (5'-GTGYCAGCMGCCGCGGTAA) and 926R (5'-CCGYCAATTYMTTTRAGTTT) and were created using a 97% similarity cutoff 
## Note: These populations have been previously subsampled to the smallest library size
## Note: All of the above processing took place in mothur
data<-read.table("MB.0.03.subsample.fn.shared",header=T,na.strings="NA")

## Get rid of first three columns since the OTUs don't actually start until the 4th column
data1 = data[-1][-1][-1]

## You should turn your raw abundance values into a relative abundance matrix and transform it
## I recommend a Hellinger Transformation (a square root of the realtive abundance)--this effectively gives low weight to variables with low counts and many zeros
## It would also also be possible to do the Logarithmic tranformation of Anderson et al. (2006): log_b(x)+1 for X>0 and zeros are left as zeros
library("vegan", lib.loc="~/R/win-library/3.3")
HellingerData<-decostand(data1,method = "hellinger")
#AndersonData<-decostand(data1,method = "log", logbase=2)

## You have to limit the OTUs to the most frequent ones (ones that occur in multiple samples so that you can measure co-occurance across samples)
## I just looked at my data file and looked for where zeros became extremely common. This was easy because mothur sorts the OTUs accroding to abundance
## If you would like a more objective way of selecting the OTUs or if your OTUs are not sorted you then this code may help:
## lessdata <- data1[,colSums(data1) > 0.05] ##Though you will have to decide what cutoff works best for your data
## You also have to reattach Group Name column
RelAbun1 = data.frame(data[2],HellingerData[1:750])

## Write file
write.table(RelAbun1, file = "MontereyRelAbun.txt", sep="\t")


##################################################################################
##################################################################################
##################################################################################
## WGCNA: Weighted Gene Correlation Network Analysis
## There is a lot of great documentation online by the authors (https://horvath.genetics.ucla.edu/html/CoexpressionNetwork/Rpackages/WGCNA/Tutorials/)
## The following shows my application of it to 16S rRNA abundance data I had


library("WGCNA", lib.loc="~/R/win-library/3.3")

## Note: If you see gene is written it actually means taxon or OTU

##################################################################################

## 1 Start and visualizing data

## Bring data in:
OTUs<-read.table("MontereyRelAbun.txt",header=T,sep="\t")
dim(OTUs);
names(OTUs);

## Turn the first column (sample name) into row names (so that only OTUs make up actual columns):
datExpr0 = as.data.frame((OTUs[,-c(1)]));
names(datExpr0) = names(OTUs)[-c(1)];
rownames(datExpr0) = OTUs$Group;

##################################################################################

## Check Data for excessive missingness:
gsg = goodSamplesGenes(datExpr0[-1], verbose = 3);
gsg$allOK
## TRUE if all OTUs have passed the cut
## This means that when you limited your OTUs to the most common OTUs above that you didn't leave any in that had too many zeros
## It is still possible that you were too choosy though
## If you got FALSE here then follow the steps below:

##################################################################################
## Did not need to do following step with Monterey

## If false then remove the offenders:
#if (!gsg$allOK)
#{
#  # Optionally, print the OTU and sample names that were removed:
#  if (sum(!gsg$goodGenes)>0)
#    printFlush(paste("Removing genes:", paste(names(datExpr0)[!gsg$goodGenes], collapse = ", ")));
#  if (sum(!gsg$goodSamples)>0)
#    printFlush(paste("Removing samples:", paste(rownames(datExpr0)[!gsg$goodSamples], collapse = ", ")));
#  # Remove the offending genes and samples from the data:
#  datExpr0 = datExpr0[gsg$goodSamples, gsg$goodGenes]
#}

##################################################################################

## Cluster the samples to see if there are any obvious outliers
sampleTree = hclust(dist(datExpr0), method = "average");

sizeGrWindow(12,9)

par(cex = 0.6);
par(mar = c(0,4,2,0))

plot(sampleTree, main = "Sample clustering to detect outliers", sub="", xlab="", cex.lab = 1.5,
     cex.axis = 1.5, cex.main = 2)
## The sample dendrogram doesn't show any obvious outliers so I didn't remove any samples. If you need to remove some samples the code is below.

##################################################################################
## No outliers for Monterey so didn't have to do

## Can remove outlier by changing height cut (could also go in and remove it mannually)
## Plot a line to show the cut:
#abline(h = 15, col = "red");

## Determine cluster under the line:
#clust = cutreeStatic(sampleTree, cutHeight = 15, minSize = 10)
#table(clust)

## clust 1 contains the samples we want to keep:
#keepSamples = (clust==1)
#datExpr = datExpr0[keepSamples, ]
#nTaxa = ncol(datExpr)
#nSamples = nrow(datExpr)

## Note: in subsequent analyses you will have to change datExpr0 to datExpr
##################################################################################

## Now read in trait (Environmental) data and match with expression samples:
traitData = read.csv("OxygenMatrixMonterey.csv");
dim(traitData)
names(traitData)

## Form a data frame analogous to expression data (relative abundances of OTUs) that will hold the Environmental traits:
## Note: used datExpr0 rather than datExpr because we did not have to ditch any samples above:
OTUSamples = rownames(datExpr0);
traitRows = match(OTUSamples, traitData$Sample);
datTraits = traitData[traitRows, -1];
rownames(datTraits) = traitData[traitRows, 1];
collectGarbage()

## Now expression data (relative abundances of OTUs) is in the variable datExpr0 and the corresponding Environmental traits in the variable datTraits

##################################################################################
## Visualize how Environmental traits relate to culstered samples (visualized with the dendogram from above)

## Again, used datExpr0 for this because we did not need to do certain above steps
## Can use any of the datTraits for this step

## Re-cluster samples:
sampleTree2 = hclust(dist(datExpr0), method = "average")
## Convert traits to a color representation: white means low, red means high, grey means missing entry:
traitColors = numbers2colors(datTraits[5:13], signed = FALSE);
## Plot the sample dendrogram and the colors underneath:
plotDendroAndColors(sampleTree2, traitColors,
                    groupLabels = names(datTraits[5:13]),
                    main = "Sample dendrogram and trait heatmap")
## White means a low value and red means a high value. Gray means missing entry.

## Can also visualze just one trait with the dendrogram:
traitColors = numbers2colors(datTraits$UpwellingS, signed = FALSE);
plotDendroAndColors(sampleTree2, traitColors,
                    groupLabels = names(datTraits[5:13]),
                    main = "Sample dendrogram and trait heatmap")


##################################################################################

## Save
save(datExpr0, datTraits, file = "Monterey-dataInput.RData")


##################################################################################
##################################################################################
##################################################################################
## 2

## Start of network analysis

## Do this first no matter what way network is made

## The following setting is important, do not omit.
options(stringsAsFactors = FALSE);
## Allows multi-threading within WGCNA. This helps speed up certain calculations.
## At present this call is necessary for the code to work.
## Any error here may be ignored but you may want to update WGCNA if you see one.

enableWGCNAThreads()

## Load the data saved in the first part
lnames = load(file = "Monterey-dataInput.RData");

## The variable lnames contains the names of loaded variables.
lnames

##################################################################################
##################################################################################
## 2a--I did NOT do this method of creating the network for Monterey
## This is a one step method of network construction (minimal efforts but the values may not be optimized)
## For details visit the code can be found: https://horvath.genetics.ucla.edu/html/CoexpressionNetwork/Rpackages/WGCNA/Tutorials/Consensus-NetworkConstruction-blockwise.pdf

##################################################################################
##################################################################################
## 2b

## The following sections provide a step by step construction of the network and module detection:

##################################################################################

## Choosing soft-thresholding power: analysis of network topology

## Choose a set of soft-thresholding powers:
powers = c(c(1:10), seq(from = 11, to=30, by=1))

## Call the network topology analysis function:
## Note: using a signed network because it preserves the sign of the connection (whether nodes are positively or negatively correlated); this is recommendation by authors of WGCNA:
sft = pickSoftThreshold(datExpr0, powerVector = powers, verbose = 5, networkType = "signed")

## Plot the results:
sizeGrWindow(9, 5)
par(mfrow = c(1,2));
cex1 = 0.9;

## Scale-free topology fit index as a function of the soft-thresholding power:
plot(sft$fitIndices[,1], -sign(sft$fitIndices[,3])*sft$fitIndices[,2],
     xlab="Soft Threshold (power)",ylab="Scale Free Topology Model Fit,signed R^2",type="n",
     main = paste("Scale independence"));
text(sft$fitIndices[,1], -sign(sft$fitIndices[,3])*sft$fitIndices[,2],
     labels=powers,cex=cex1,col="red");

## This line corresponds to using an R^2 cut-off of h:
abline(h=0.8,col="red")

## Mean connectivity as a function of the soft-thresholding power:
plot(sft$fitIndices[,1], sft$fitIndices[,5],
     xlab="Soft Threshold (power)",ylab="Mean Connectivity", type="n",
     main = paste("Mean connectivity"))
text(sft$fitIndices[,1], sft$fitIndices[,5], labels=powers, cex=cex1,col="red")

## I picked a soft thresholding value of 10 because it was well above an r2 of 0.8 (it is a local peak for the r2) and the mean connectivity is still above 0
##################################################################################

## Calculate the adjacencies, using the soft thresholding power of ___:
## Note: Have to specify signed network again:
softPower = 10;
adjacency = adjacency(datExpr0, power = softPower, type = "signed");

##################################################################################

## Transform adjacency into Topological Overlap Matrix and calculate corresponding dissimilarity:
## Note: The TOM you calculate shows the topological similarity of nodes, factoring in the connection strength two nodes share with other "third party" nodes  
## This will minimize effects of noise and spurious associations:

## Turn adjacency into topological overlap:
## Note: Have to specify signed network again:
TOM = TOMsimilarity(adjacency, TOMType = "signed");
dissTOM = 1-TOM

##################################################################################

## Create a dendogram using a hierarchical clustering tree

## Call the hierarchical clustering function
TaxaTree = hclust(as.dist(dissTOM), method = "average");

## Plot the resulting clustering tree (dendrogram)
sizeGrWindow(12,9)
plot(TaxaTree, xlab="", sub="", main = "Taxa clustering on TOM-based dissimilarity",
     labels = FALSE, hang = 0.04);

##################################################################################

## You have to decide the optimal module size for your system and should play around with this value a little
## I wanted relatively large module so I set the minimum module size relatively high (30):
minModuleSize = 30;

## Module identification using dynamic tree cut:
dynamicMods = cutreeDynamic(dendro = TaxaTree, distM = dissTOM,
                            deepSplit = 2, pamRespectsDendro = FALSE,
                            minClusterSize = minModuleSize);
table(dynamicMods)

##################################################################################

## Convert numeric lables into colors
dynamicColors = labels2colors(dynamicMods)
table(dynamicColors)

## Plot the dendrogram with module colors underneath
sizeGrWindow(8,6)
plotDendroAndColors(TaxaTree, dynamicColors, "Dynamic Tree Cut",
                    dendroLabels = FALSE, hang = 0.03,
                    addGuide = TRUE, guideHang = 0.05,
                    main = "Gene dendrogram and module colors")


##################################################################################

## Quatify co-expression similarity of the entire modules using eigengenes and cluter them on their correlation:
## Note: An eigengene is 1st principal component of module expression matrix and represents a suitably defined average OTU community

## Calculate eigengenes
MEList = moduleEigengenes(datExpr0, colors = dynamicColors)
MEs = MEList$eigengenes

## Calculate dissimilarity of module eigengenes--Can fix if change to 1+ rather than 1- because all eigengenes were negative
MEDiss = 1-cor(MEs);

## Cluster module eigengenes
METree = hclust(as.dist(MEDiss), method = "average");

## Plot the result
sizeGrWindow(7, 6)
plot(METree, main = "Clustering of module eigengenes",
     xlab = "", sub = "")

##################################################################################

## Chose a height cut of 0.25, corresponding to correlation of 0.75, to merge--so nothing got merged
MEDissThres = 0.30

## Plot the cut line into the dendrogram
abline(h=MEDissThres, col = "red")

## Call an automatic merging function
merge = mergeCloseModules(datExpr0, dynamicColors, cutHeight = MEDissThres, verbose = 3)

## The merged module colors
mergedColors = merge$colors;

## Eigengenes of the new merged modules:
mergedMEs = merge$newMEs;

##################################################################################

## If you had combined different modules then that would show in this plot:

sizeGrWindow(12, 9)

plotDendroAndColors(TaxaTree, cbind(dynamicColors, mergedColors),
                    c("Dynamic Tree Cut", "Merged dynamic"),
                    dendroLabels = FALSE, hang = 0.03,
                    addGuide = TRUE, guideHang = 0.05)

##################################################################################

## Rename to moduleColors
moduleColors = mergedColors

## Construct numerical labels corresponding to the colors
colorOrder = c("grey", standardColors(50));
moduleLabels = match(moduleColors, colorOrder)-1;
MEs = mergedMEs;

## Save module colors and labels for use in subsequent parts
save(MEs, moduleLabels, moduleColors, TaxaTree, file = "Monterey-networkConstruction-stepByStep.RData")

####################################################################################
## Extra Visualization--Can due after modules are assigned

cmd1=cmdscale(as.dist(dissTOM),2)
par(mfrow=c(1,1)) 
plot(cmd1, col=as.character(dynamicColors),  main="
     MDS plot",xlab="Scaling 
     Dimension 1",ylab="Scaling Dimension 2",
     cex.axis=1.5,cex.lab=1.5, cex.main=1.5) 

## Vs. your merged modules (this will look identical to the first MDS plot since we didn't merge any modules):
cmd1=cmdscale(as.dist(dissTOM),2)
par(mfrow=c(1,1)) 
plot(cmd1, col=as.character(mergedColors),  main="
     MDS plot",xlab="Scaling 
     Dimension 1",ylab="Scaling Dimension 2",
     cex.axis=1.5,cex.lab=1.5, cex.main=1.5) 


##################################################################################
##################################################################################
## 2C--Step 2 for big datasets. I haven't had to do this method yet, but the code can be found: https://horvath.genetics.ucla.edu/html/CoexpressionNetwork/Rpackages/WGCNA/Tutorials/Consensus-NetworkConstruction-blockwise.pdf


##################################################################################
##################################################################################
##################################################################################
## 3

## Relating modules to external information and IDing important taxa

##################################################################################

## Identify modules that are significantly associated with the measured Environmental traits:

## You already have summary profiles for each module (eigengenes), so we just have to correlate these eigengenes with Environmental traits and look for significant associations:
## First, define numbers of OTUs and samples
nTaxa = ncol(datExpr0);
nSamples = nrow(datExpr0);

## Recalculate MEs (modules) with color labels
MEs0 = moduleEigengenes(datExpr0, moduleColors)$eigengenes
MEs = orderMEs(MEs0)
moduleTraitCor = cor(MEs, datTraits, use = "p");
moduleTraitPvalue = corPvalueStudent(moduleTraitCor, nSamples);

## Now visualize it:
sizeGrWindow(10,6)

## Will display correlations and their p-values
textMatrix = paste(signif(moduleTraitCor, 2), "\n(",
                   signif(moduleTraitPvalue, 1), ")", sep = "");
dim(textMatrix) = dim(moduleTraitCor)
par(mar = c(6, 8.5, 3, 3));

## Display the correlation values within a heatmap plot
labeledHeatmap(Matrix = moduleTraitCor,
               xLabels = names(datTraits),
               yLabels = names(MEs),
               ySymbols = names(MEs),
               colorLabels = FALSE,
               colors = greenWhiteRed(50),
               textMatrix = textMatrix,
               setStdMargins = FALSE,
               cex.text = 0.5,
               zlim = c(-1,1),
               main = paste("Module-trait relationships"))

## Each row corresponds to a module eigengene, column to an Environmental trait. Each cell contains the corresponding Pearson correlation coefficient (top number) and p-vlaue (in parentheses).
## Table is color-coded by correlation according to the color legend.

##################################################################################

## This section quantifies associations of individual taxa with out trait of interest
## Have to look at previous step to figure out what color modules correlate significantly with certain traits

## Define variable weight containing the weight column of datTrait
CR = as.data.frame(datTraits$CR);
names(CR) = "CR"

## names (colors) of the modules
modNames = substring(names(MEs), 3)
TaxaModuleMembership = as.data.frame(cor(datExpr0, MEs, use = "p"));
MMPvalue = as.data.frame(corPvalueStudent(as.matrix(TaxaModuleMembership), nSamples));
names(TaxaModuleMembership) = paste("MM", modNames, sep="");
names(MMPvalue) = paste("p.MM", modNames, sep="");
TaxaTraitSignificance = as.data.frame(cor(datExpr0, CR, use = "p"));
GSPvalue = as.data.frame(corPvalueStudent(as.matrix(TaxaTraitSignificance), nSamples));
names(TaxaTraitSignificance) = paste("GS.", names(CR), sep="");
names(GSPvalue) = paste("p.GS.", names(CR), sep="");

##################################################################################

module = "red"
column = match(module, modNames);
moduleTaxa = moduleColors==module;
sizeGrWindow(7, 7);
par(mfrow = c(1,1));
verboseScatterplot(abs(TaxaModuleMembership[moduleTaxa, column]),
                   abs(TaxaTraitSignificance[moduleTaxa, 1]),
                   xlab = paste("Module Membership in", module, "module"),
                   ylab = "Taxa significance for CR",
                   main = paste("Module membership vs. Taxa significance\n"),
                   cex.main = 1.2, cex.lab = 1.2, cex.axis = 1.2, col = module)

## This graph shows you each taxa that made it into the "red" module and how each taxa correlated with 1) the Environmental trait of interest and 2) how important it is to the module
## The taxa/OTUs that have high module membership tend to occur whenever the module is represented in the environment and are therefore often connected throughout the samples with other red taxa/OTUs

##################################################################################

## Merge the statistical info from previous section (modules with high assocation with trait of interest--e.g. CR or Temp) with taxa annotation and write a file that summarizes these results

names(datExpr0)
names(datExpr0)[moduleColors=="red"]

## Need to figure out proper file for TaxaAnnotation
annot = read.table("MB.subsample.fn.0.03.cons.taxonomy",header=T,sep="\t");
dim(annot)
names(annot)
probes = names(datExpr0)
probes2annot = match(probes, annot$OTU)

## The following is the number or probes without annotation:
sum(is.na(probes2annot))
## Should return 0.

##################################################################################

## Tempeate the starting data frame
TaxaInfo0 = data.frame(Taxon = probes,
                       TaxaSymbol = annot$OTU[probes2annot],
                       LinkID = annot$Taxonomy[probes2annot],
                       moduleColor = moduleColors,
                       TaxaTraitSignificance,
                       GSPvalue)

## Order modules by their significance for weight
modOrder = order(-abs(cor(MEs, CR, use = "p")));

## Add module membership information in the chosen order
for (mod in 1:ncol(TaxaModuleMembership))
{
  oldNames = names(TaxaInfo0)
  TaxaInfo0 = data.frame(TaxaInfo0, TaxaModuleMembership[, modOrder[mod]],
                         MMPvalue[, modOrder[mod]]);
  names(TaxaInfo0) = c(oldNames, paste("MM.", modNames[modOrder[mod]], sep=""),
                       paste("p.MM.", modNames[modOrder[mod]], sep=""))
}

## Order the OTUs in the geneInfo variable first by module color, then by geneTraitSignificance
TaxaOrder = order(TaxaInfo0$moduleColor, -abs(TaxaInfo0$GS.CR));
TaxaInfo = TaxaInfo0[TaxaOrder, ]

##################################################################################

## Write file

write.csv(TaxaInfo, file = "TaxaInfo.csv")

## NOTES:
## GS stands for Gene Significance (for us it means taxon significance) while MM stands for module membership
## MM column gives the module membership correlation (correlates expression with module eigengene of a given module). If close to 0 then the gene/taxa is not part of that color module. If it is close to 1 or -1 then it is highly connected to that color module genes/taxa.
## GS allows incorporation of extrnal info int he co-expression network by showing gene significance. The higher the absolute value of GS the more biologically significant the ... gene/taxa.
## Modules will be ordered by their signifcance for weight, with the most signifcant ones to the left.
## Each of the modules (with each OTU assigned to exactly one module) will be represented for the Environmental trait you selected
## You will have to rerun this for each Environmental trait you are interested in

## Output: moduleColor = module that the OTU was ultimately put in
## Output: GS.Environmentaltrait = Pearson Correlation Coefficient for that OTU with the trait
## Output: p.GS.Environmentaltrait = P value for the preceeding relationship
## Output: MM.color = Pearson Correlation Coefficient for Module Membership--i.e. how well that OTU correlates with that particular color module (each OTU has a value for each module but only belongs to one module)
## Output: p.MM.color = P value for the preceeding relationship


##################################################################################
##################################################################################
## 5a--Visualizing the OTU/taxa network

nOTU = ncol(datExpr0)
nSamples = nrow(datExpr0)

## Calculate topological overlap anew: this could be done more efficiently by saving the TOM
## calculated during module detection, but let us do it again here.
dissTOM = 1-TOMsimilarityFromExpr(datExpr0, power = 10, networkType = "signed");

## Transform dissTOM with a power to make moderately strong connections more visible in the heatmap
plotTOM = dissTOM^10;

## Set diagonal to NA for a nicer plot
diag(plotTOM) = NA;

## Call the plot function
sizeGrWindow(9,9)
TOMplot(plotTOM, TaxaTree, moduleColors, main = "Network heatmap plot, all OTUs")

##################################################################################
## If normal 5a heatmap generation takes too long you can restrict the number of OTU/taxa
## Still have to do first few steps from above (dissTOM creation)

nSelect = 200

## For reproducibility, we set the random seed
set.seed(10);
select = sample(nOTU, size = nSelect);
selectTOM = dissTOM[select, select];

## There's no simple way of restricting a clustering tree to a subset of OTU, so we must re-cluster.
selectTree = hclust(as.dist(selectTOM), method = "average")
selectColors = moduleColors[select];

## Open a graphical window
sizeGrWindow(9,9)

## Taking the dissimilarity to a power, say 10, makes the plot more informative by effectively changing
## the color palette; setting the diagonal to NA also improves the clarity of the plot
plotDiss = selectTOM^10;
diag(plotDiss) = NA;
TOMplot(plotDiss, selectTree, selectColors, main = "Network heatmap plot, selected OTU")

##################################################################################
