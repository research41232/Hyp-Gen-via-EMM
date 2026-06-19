
# Does-GenAI-Help
<img width="1024" height="768" alt="graphicalabstractnew" src="https://github.com/user-attachments/assets/a0156e97-b40d-4343-bda1-a996d3e08ba7" />

This repository is built for the paper
*Why "Does GenAI Help?" Is the Wrong Question: Effects Change Sign Across Student Subgroups* .

## Overview
In this work, we instantiate PLS-SEM as a target model class within the EMM framework to discover student subgroups whose structural relationships between GenAI use and academic performance differ from the overall population. To improve interpretability, we complement EMM with a contrastive analysis that constructs minimally different comparison groups, enabling practitioners to understand which conditions drive the observed deviations.

As the first wave of a longitudinal study, we collected data in a large introductory data science course enrolling more than 1,000 first-year bachelor students across multiple study programs. We combine in-situ survey data on GenAI attitudes, skills, and usage behavior with exam outcomes to study how associations between GenAI usage modes and academic performance vary across students.

Here, we provide the [full surveys](surveys/), the [construct specifications](constructs/), and relevant parts of our source code. To deploy EMM, we extend the [pysubgroup](https://github.com/flemmerich/pysubgroup) (Lemmerich & Becker, 2018) package with an instantiation of [PLS-SEM as a target model class](PLS-SEM-extension/). This extension can be used on any dataset and with any PLS-SEM structure, provided that its configuration is specified using the [plspm](https://pypi.org/project/plspm/) package.

## Prerequisites & Installation
Install [pysubgroup](https://github.com/flemmerich/pysubgroup) and [plspm](https://pypi.org/project/plspm/), using pip:
```
pip install pysubgroup
pip install plspm
``` 
Note: for some users the pip installation of [pysubgroup](https://github.com/flemmerich/pysubgroup) fails. If that is the case, solutions are provided in their repository. 

Once installed:
1. Download the files in the [PLS-SEM-extension](PLS-SEM-extension/) folder;
2. Locate the folder `pysubgroup` within your `site-packages` directory;
3. Delete the `__init__.py` file;
4. Move the downloaded files to the `pysubgroup` folder;
5. You can now import the extended package with `import pysubgroup`.
## How To Use
1. Define your constructs. All items belonging to a construct must share a common prefix and and be the same .Scale (if nonmetric). 
2. Define your PLS-SEM configuration. This configuration will be the same for the global model and all discovered exceptional models, and must be specified using  the [plspm](https://pypi.org/project/plspm/) package. In this work, we define the following configuration:
```
import plspm.config as c
from plspm.plspm import Plspm
from plspm.scheme import Scheme
from plspm.mode import Mode
from plspm.scale import Scale

# define model structure in terms of latent constructs
structure = c.Structure()
structure.add_path(["Embracing_GenAI"], ["GenAI_Skill", "Study_Aid_Use", "Improve_Use", "Generate_Use"])
structure.add_path(["GenAI_Skill"], ["Study_Aid_Use", "Improve_Use", "Generate_Use"])
structure.add_path(["Study_Aid_Use", "Improve_Use", "Generate_Use"], ["Perceived_Learning_Impact"])
structure.add_path(["Study_Aid_Use", "Improve_Use", "Generate_Use"], ["Exam_Grade"])
structure.add_path(["Perceived_Learning_Impact"], ["Exam_Grade"])

config = c.Config(structure.path(), default_scale=Scale.NUM)

# couple each latent construct with its respective items
# all items corresponding to a construct must share a common prefix (e.g., "embracing_")
# and be the same .Scale (if nonmetric).

config.add_lv_with_columns_named("Embracing_GenAI", Mode.A, df, "embracing_")
config.add_lv_with_columns_named("GenAI_Skill", Mode.A, df, "skill_")
config.add_lv_with_columns_named("Study_Aid_Use", Mode.A, df, "study_aid_")
config.add_lv_with_columns_named("Improve_Use", Mode.A, df, "improve_")
config.add_lv_with_columns_named("Generate_Use", Mode.A, df, "generate_")
config.add_lv_with_columns_named("Perceived_Learning_Impact", Mode.A, df, "impact_")
config.add_lv_with_columns_named("Exam_Grade", Mode.A, df, "grade")
```
3. Fit and inspect the global PLS-SEM using:
```
pls = Plspm(df, config, Scheme.PATH, 100, 0.00000001, False)
pls.inner_model()
```
4. Use EMM to discover interpretable subgroups where the structural model deviates from the global pattern: 
    1. Select features to include in the descriptive space. In this work, we included all demographics and survey items that were not part of a latent construct;
    2. Choose an appropriate depth (the maximum number of conditions that can describe a subgroup);
    3. Choose an appropriate quality function and set the desired parameters. We provide a choice of several quality functions in [SEM_model_target.py](PLS-SEM-extension/SEM_model_target.py). In this work, we employ `SEMQFEntropy`, which rewards (1) paths that are not statistically significant in the base model but become significant in the subgroup model, and (2) paths that are significant in both models but change sign between the base and subgroup models. Subgroup size is moderated by an entropy measure. 
```
import pysubgroup as ps
all_cols = df.columns.tolist()
descriptive_features = ['Gender', 'Major', 'Age', ...] # list all desired descriptive features
ignore = [col for col in all_cols if col not in descriptive_features]
target = ps.SEMTarget(config)
searchspace = ps.create_selectors(df, ignore=ignore)
task = ps.SubgroupDiscoveryTask (
    data,
    target,
    searchspace,
    result_set_size=5,
    depth=5,
    qf=ps.SEMQFEntropy(config, weight_sig=1, weight_sign=1))
result = ps.DFS().execute(task)
result_df = result.to_dataframe()
```
