# MSSMvsSMRun2Legacy
Analysis specific software for MSSM (with SM categories) to be used within CombineHarvester

# Setup software

```bash
wget https://raw.githubusercontent.com/KIT-CMS/MSSMvsSMRun2Legacy/master/scripts/checkout.sh
bash ./checkout.sh
```
After this, please put the datacard inputs into the [shapes](shapes) folder in the following structure:

`<year>/<channel>/htt_<category>.inputs-mssm-vs-sm-Run<year>-<variable>.root`

# Cut-based SM analysis

## Datacard creation

```bash
morph_parallel.py --output output --analysis sm --eras 2016,2017,2018 --category_list ${CMSSW_BASE}/src/CombineHarvester/MSSMvsSMRun2Legacy/input/control_region_categories.txt --variable mt_tot_puppi --parallel 5
morph_parallel.py --output output --analysis sm --eras 2016,2017,2018 --category_list ${CMSSW_BASE}/src/CombineHarvester/MSSMvsSMRun2Legacy/input/sm_categories.txt --variable m_sv_puppi --parallel 5

for era in 2016 2017 2018;
do
    mkdir -p output_sm/${era}/cmb/; rsync -av --progress output_sm/${era}/htt_*/*  output_sm/${era}/cmb/
done;
mkdir -p output_sm/combined/cmb/; rsync -av --progress output_sm/201?/htt_*/*  output_sm/combined/cmb/
```

## Workspace creation

**inclusive and stage 0:**

```bash
ulimit -s unlimited
combineTool.py -M T2W -i output_sm/{2016,2017,2018,combined}/cmb --parallel 4 -o inclusive.root
combineTool.py -M T2W -i output_sm/{2016,2017,2018,combined}/cmb --parallel 4 -o stage0.root -P HiggsAnalysis.CombinedLimit.PhysicsModel:multiSignalModel --PO '"map=^.*/ggH125.?$:r_ggH[1,-9,11]"' --PO '"map=^.*/qqH125.?$:r_qqH[1,-9,11]"'
```

## Signal strength scans

**inclusive:**

```bash
combineTool.py -M MultiDimFit -d output_sm/combined/cmb/inclusive.root --there --algo grid --robustFit 1 --X-rtd MINIMIZER_analytic --cminDefaultMinimizerStrategy 0 --cminDefaultMinimizerTolerance 0.01 --floatOtherPOIs 1 -t -1 --expectSignal 1 -v1 --alignEdges 1 --setParameterRanges r=0.85,1.15 --points 41 --split-points 1 -n .inclusive --job-mode condor --task-name r_1Dscan --dry-run

# after adapting the condor configuration, submit to batch system:
condor_submit condor_r_1Dscan.sub
```

**stage 0:**

```bash
# ggH
combineTool.py -M MultiDimFit -d output_sm/combined/cmb/stage0.root --there --algo grid --robustFit 1 --X-rtd MINIMIZER_analytic --cminDefaultMinimizerStrategy 0 --cminDefaultMinimizerTolerance 0.01 --floatOtherPOIs 1 -t -1 --expectSignal 1 -v1 --setParameters r_ggH=1,r_qqH=1 --redefineSignalPOIs r_ggH --alignEdges 1 -M MultiDimFit -d stage0.root --setParameterRanges r_ggH=0.65,1.35:r_qqH=0.3,1.7 --points 41 --split-points 1 -n .stage0_r_ggH --job-mode condor --task-name r_ggH_1Dscan --dry-run

# qqH
combineTool.py -M MultiDimFit -d output_sm/combined/cmb/stage0.root --there --algo grid --robustFit 1 --X-rtd MINIMIZER_analytic --cminDefaultMinimizerStrategy 0 --cminDefaultMinimizerTolerance 0.01 --floatOtherPOIs 1 -t -1 --expectSignal 1 -v1 --setParameters r_ggH=1,r_qqH=1 --redefineSignalPOIs r_qqH --alignEdges 1 -M MultiDimFit -d stage0.root --setParameterRanges r_qqH=0.65,1.35:r_ggH=0.3,1.7 --points 41 --split-points 1 -n .stage0_r_qqH --job-mode condor --task-name r_qqH_1Dscan --dry-run

# after adapting the condor configuration, submit to batch system:
condor_submit condor_r_ggH_1Dscan.sub
condor_submit condor_r_qqH_1Dscan.sub
```

**Collecting results & plotting likelihood scans:**

```bash
hadd output_sm/combined/cmb/higgsCombine.inclusive.MultiDimFit.mH120.root output_sm/combined/cmb/higgsCombine.inclusive.POINTS*.root
hadd output_sm/combined/cmb/higgsCombine.stage0_r_ggH.MultiDimFit.mH120.root output_sm/combined/cmb/higgsCombine.stage0_r_ggH.POINTS*.root
hadd output_sm/combined/cmb/higgsCombine.stage0_r_qqH.MultiDimFit.mH120.root output_sm/combined/cmb/higgsCombine.stage0_r_qqH.POINTS*.root

plot1DScan.py output_sm/combined/cmb/higgsCombine.inclusive.MultiDimFit.mH120.root -o r_1Dscan
plot1DScan.py output_sm/combined/cmb/higgsCombine.stage0_r_ggH.MultiDimFit.mH120.root -o r_ggH_1Dscan --POI r_ggH
plot1DScan.py output_sm/combined/cmb/higgsCombine.stage0_r_qqH.MultiDimFit.mH120.root -o r_qqH_1Dscan --POI r_qqH
```

## Prefit shapes

```bash
for card in output_MSSMvsSM_Run2_sm_nobtag_m_sv_puppi/*/cmb/combined.txt.cmb;
do
    PostFitShapesFromWorkspace -w ${card/combined.txt.cmb/stage0.root}  -o ${card/combined.txt.cmb/prefit_shapes.root} -d ${card}
done;
```

# Model-independent MSSM analysis

## Datacard creation

```bash
morph_parallel.py --output output --analysis mssm --eras 2016,2017,2018 --category_list ${CMSSW_BASE}/src/CombineHarvester/MSSMvsSMRun2Legacy/input/control_region_categories.txt --variable mt_tot_puppi --additional_arguments "--auto_rebin=1 ${CMSSW_BASE}/src/CombineHarvester/MSSMvsSMRun2Legacy/data/higgs_pt_v3.root" --parallel 5
morph_parallel.py --output output --analysis mssm --eras 2016,2017,2018 --category_list ${CMSSW_BASE}/src/CombineHarvester/MSSMvsSMRun2Legacy/input/mssm_classic_categories.txt --variable mt_tot_puppi --additional_arguments "--auto_rebin=1 ${CMSSW_BASE}/src/CombineHarvester/MSSMvsSMRun2Legacy/data/higgs_pt_v3.root" --parallel 5

for era in 2016 2017 2018;
do
    mkdir -p output_mssm/${era}/cmb/; rsync -av --progress output_mssm/${era}/htt_*/*  output_mssm/${era}/cmb/
done;
mkdir -p output_mssm/combined/cmb/; rsync -av --progress output_mssm/201?/htt_*/*  output_mssm/combined/cmb/
```

## Workspace creation

```bash
ulimit -s unlimited
combineTool.py -M T2W -o "ws.root" -P HiggsAnalysis.CombinedLimit.PhysicsModel:multiSignalModel --PO '"map=^.*/ggh_(i|t|b).?$:r_ggH[0,0,200]"' --PO '"map=^.*/bbh$:r_bbH[0,0,200]"' -i output_mssm/{2016,2017,2018,combined}/cmb -m 110 --parallel 4
```

## Model-independent CLs 95% limits (asymptotic, SM Higgs in background hypothesis)

**bbH:**

```bash
combineTool.py -m "110,120,130,140,160,180,200,250,300,350,400,450,500,600,700,800,900,1000,1200,1400,1600,1800,2000,2300,2600,2900,3200" -M AsymptoticLimits --rAbsAcc 0 --rRelAcc 0.0005 --boundlist CombineHarvester/MSSMvsSMRun2Legacy/input/mssm_boundaries.json --setParameters r_ggH=0,r_bbH=0 --redefineSignalPOIs r_bbH -d output_mssm/combined/cmb/ws.root --there -n ".bbH" --job-mode condor --dry-run --task-name bbH_full_combined --X-rtd MINIMIZER_analytic --cminDefaultMinimizerStrategy 0 --cminDefaultMinimizerTolerance 0.01 -v 1

# After adaption of condor_bbH_full_combined.sub, submit to batch system:
condor_submit condor_bbH_full_combined.sub
```

**ggH:**

```bash
combineTool.py -m "110,120,130,140,160,180,200,250,300,350,400,450,500,600,700,800,900,1000,1200,1400,1600,1800,2000,2300,2600,2900,3200" -M AsymptoticLimits --rAbsAcc 0 --rRelAcc 0.0005 --boundlist CombineHarvester/MSSMvsSMRun2Legacy/input/mssm_boundaries.json --setParameters r_ggH=0,r_bbH=0 --redefineSignalPOIs r_ggH -d output_mssm/combined/cmb/ws.root --there -n ".ggH" --job-mode condor --dry-run --task-name ggH_full_combined --X-rtd MINIMIZER_analytic --cminDefaultMinimizerStrategy 0 --cminDefaultMinimizerTolerance 0.01 -v 1

# After adaption of condor_ggH_full_combined.sub, submit to batch system:
condor_submit condor_ggH_full_combined.sub
```

**Collecting limits:**

```bash
for p in gg bb
do
    combineTool.py -M CollectLimits output_mssm/combined/cmb/higgsCombine.${p}H*.root --use-dirs -o output_mssm/combined/cmb/mssm_${p}H_combined.json
done
```

**Plotting limits:**

```bash
for p in gg bb
do
    plotMSSMLimits.py --cms-sub "Own Work" --title-right "137 fb^{-1} (13 TeV)" --process "${p}#phi" --y-axis-min 0.0001 --y-axis-max 1000.0 --show exp,obs output_mssm/combined/cmb/mssm_${p}H_combined_cmb.json  --output mssm_model-independent_combined_${p}H_cmb --logx --logy
done
```

# Model-dependent MSSM analysis with standard categorization

## Datacard creation

```bash
 MorphingMSSMvsSM --era=2017 --auto_rebin=1 --binomial_bbb=1 --variable=mt_tot_puppi --analysis=mssm_vs_sm_standard
```

## Workspace creation

```bash
for model in mh125 mh125_ls mh125_lc mh125_align
do
    combineTool.py -M T2W -o ws_${model}.root  -P CombineHarvester.MSSMvsSMRun2Legacy.MSSMvsSM:MSSMvsSM --PO filePrefix=${CMSSW_BASE}/src/CombineHarvester/MSSMvsSMRun2Legacy/data/ --PO modelFile=13,Run2017,${model}_13.root --PO MSSM-NLO-Workspace=${CMSSW_BASE}/src/CombineHarvester/MSSMvsSMRun2Legacy/data/higgs_pt_v3_mssm_mode.root -i output_MSSMvsSM_Run2_mssm_vs_sm_standard_mt_tot_puppi/Run2017/cmb/ --PO minTemplateMass=100.0 --PO maxTemplateMass=3200.0
done
```

## Model-dependent CLs 95% limits (asymptotic, SM as signal in 0-hypothesis)

**Computing limits:**

```bash
for model in mh125 mh125_ls mh125_lc mh125_align
do
    combineTool.py -M AsymptoticGrid CombineHarvester/MSSMvsSMRun2Legacy/input/mssm_asymptotic_grid_${model}.json -d output_MSSMvsSM_Run2_mssm_vs_sm_standard_mt_tot_puppi/Run2017/cmb/ws_${model}.root --job-mode 'condor' --task-name 'mssm_${model}' --dry-run --redefineSignalPOI x --setParameters r=1 --freezeParameters r -v1
done

# After adaption of each shell script and condor configuration matching mattern condor_mssm_${model}.{sh,sub}, submit to batch system:
for model in mh125 mh125_ls mh125_lc mh125_align
do
    condor_submit  condor_mssm_${model}.sub
done
```

**Collecting limits:**

```bash
for model in mh125 mh125_ls mh125_lc mh125_align
do
    combineTool.py -M AsymptoticGrid CombineHarvester/MSSMvsSMRun2Legacy/input/mssm_asymptotic_grid_${model}.json -d output_MSSMvsSM_Run2_mssm_vs_sm_standard_mt_tot_puppi/Run2017/cmb/ws_${model}.root --job-mode 'condor' --task-name 'mssm_${model}' --dry-run --redefineSignalPOI x --setParameters r=1 --freezeParameters r -v1; mv asymptotic_grid.root asymptotic_grid_${model}.root
done
```

**Plotting limits:**

```bash
plotLimitGrid.py asymptotic_grid_mh125.root --scenario-label="M_{h}^{125} scenario (asimov)" --output mssm_2017_mh125_asymptotic --title-right="41.5 fb^{-1} (13 TeV)" --cms-sub="Private Work" --contours="exp-2,exp-1,exp0,exp+1,exp+2,obs" --model_file=${CMSSW_BASE}/src/CombineHarvester/MSSMvsSMRun2Legacy/data/mh125_13.root
plotLimitGrid.py asymptotic_grid_mh125_lc.root --scenario-label="M_{h}^{125}(#tilde#chi) scenario (asimov)" --output mssm_2017_mh125_lc_asymptotic --title-right="41.5 fb^{-1} (13 TeV)" --cms-sub="Private Work" --contours="exp-2,exp-1,exp0,exp+1,exp+2,obs" --model_file=${CMSSW_BASE}/src/CombineHarvester/MSSMvsSMRun2Legacy/data/mh125_lc_13.root
plotLimitGrid.py asymptotic_grid_mh125_ls.root --scenario-label="M_{h}^{125}(#tilde#tau) scenario (asimov)" --output mssm_2017_mh125_ls_asymptotic --title-right="41.5 fb^{-1} (13 TeV)" --cms-sub="Private Work" --contours="exp-2,exp-1,exp0,exp+1,exp+2,obs" --model_file=${CMSSW_BASE}/src/CombineHarvester/MSSMvsSMRun2Legacy/data/mh125_ls_13.root
plotLimitGrid.py asymptotic_grid_mh125_align.root --scenario-label="M_{h}^{125}(alignment) scenario (asimov)" --output mssm_2017_mh125_align_asymptotic --title-right="41.5 fb^{-1} (13 TeV)" --cms-sub="Private Work" --contours="exp-2,exp-1,exp0,exp+1,exp+2,obs" --model_file=${CMSSW_BASE}/src/CombineHarvester/MSSMvsSMRun2Legacy/data/mh125_align_13.root
```
