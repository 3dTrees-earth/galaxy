# 3DTrees Tool integration guide

This guide explains how to integrate new tools into the 3DTrees Galaxy backend. There are two levels of integration: **Tool-Only** and **Full Integration**.
We suggest that you follow the **Tool-Only** steps, until the standalone tool runs without errors.
Then, you can either reach out to 3DTrees core-developer team to do the full integration, or you give it a try yourself and add your contribution via PR on [3DTrees](https://github.com/3dtrees-earth/3dtrees).

## Prerequisites

You need Docker for both integrations and the Python package `planemo` for the full integration.
We suggest you use macOS or Linux, as Docker runs a bit smoother here. For Windows, we suggest you move to Windows subsystem for Linux (https://learn.microsoft.com/en-us/windows/wsl/install). 
Then you can install [Docker Desktop](https://docs.docker.com/desktop/), which is preferred over the docker community edition. 

Planemo can be installed using `pip install planemo`. We suggest to use a dedicated Python environment for this. You can use conda, venv or pyenv for that. Or not.

## Tool-Only Integration

Tool-only integration means creating a Docker container, that works independently, without Galaxy. The required interface is defined in `/src/parameters.py` inside the tool repo.

### Steps for Tool-Only Integration

1. **Create Tool Structure**
Add a new repository in [3DTrees Organization](https://github.com/3dtrees-earth/).
This repository includes a self-contained version of the new tool. Galaxy will handle data and parameter input and mount defined input files into the container at runtime.
During development, we will replicate this structure. The following structure is *suggested*:

```plain
src/
│   ├── parameters.py
│   └── run.py
in/
out/

```

2. **Create Dockerfile**

You need to create the full environment for the new tool. You can use the Python evnvironment of the
overviews tool as a starting point. It already includes open3D and all its dependencies.
   
```Dockerfile
   FROM python:3.11

   RUN apt-get update && apt-get install -y \
       libgl1-mesa-glx \
       libegl1 libgl1 libgomp1

   RUN pip install \
       numpy==1.23.5 \
       open3d==0.18.0 \
       pydantic \
       pydantic-settings \
       tqdm \
       # Add your specific dependencies

   ENV EGL_PLATFORM=surfaceless

   RUN mkdir -p /src && mkdir -p /in && mkdir -p /out
   COPY ./src /src

   WORKDIR /src
   CMD ["python", "run.py"]
```

3. **Create Parameters Class**

The interface into the outside world is created in two steps: 1) a `pydantic_settings.BaseSettings` implementation, which defines all necessary parameters, and 2) an endpoint to invoke the tool, making use of these parameters.
We suggest to implement this in two files: 

3.1 `parameters.py`
```python
from pydantic_settings import BaseSettings, SettingsConfigDict
from pydantic import Field, AliasChoices
from pathlib import Path

class Parameters(BaseSettings):
    """CLI parameters for your tool"""
    dataset_path: str = Field(..., description="Input dataset path", 
                            alias=AliasChoices("dataset-path", "dataset_path"))
    output_dir: Path = Field("/out", description="Output directory",
                            alias=AliasChoices("output-dir", "output_dir"))
    # Add your specific parameters here
    
    model_config = SettingsConfigDict(
        case_sensitive=False,
        cli_parse_args=True,
        cli_ignore_unknown_args=True
    )
```

3.2. `run.py`

The main script that runs the whole tool. 

```python
import logging
from pathlib import Path
from parameters import Parameters

logger = logging.getLogger(__name__)
logger.setLevel(logging.DEBUG)

params = Parameters()
logger.info(f"Parameters: {params}")

# Your tool logic here
# Process input from params.dataset_path
# Save outputs to params.output_dir
```

4. Building the tool

There are two ways, how you can build and test the tool now. Either you use docker directly, or you set up [docker compose](https://docs.docker.com/compose/).

With docker, you build and run like:

```bash
docker build -t mytool .
docker run --rm -it -v /path/to/input-file:/in -v /path/to/outputs:/out mytool python run.py
```

This invokes the just build image and creates a container in  interactive terminal mode (`-it`). The container is deleted after it exited (`--rm`). You mount input and output folders (`-v host:container`), to persist data after the container exits. 
The command run *inside* the container is: `python run.py`.

With a `docker-compose.yml` at the root:

```yaml
services:
    mytool:
        build:
            context: .
            dockerfile: Dockerfile
        volumes:
            - ./in:/in
            - .out:/out
        command: ["python", "run.py"]
```

you can simplfy the build and run to:

```bash
docker compose up
```

5. **Test data**

You need to add test data to your project. We suggest to also create a `/in` folder in the repository and ignore all files in that folder. Then, you can add test data to that folder and it will not be uploaded to Github. 
We are working on a different approach here, to automatically grab test data from the S3 storage, but for the time being you need to add that manually.


## Full Integration

Full integration means using the Makefile to integrate the tool into a running Galaxy instance and adding unit tests to the 3dtrees-api tests. You do the steps from above as well, but you do it with a local version of the 3DTrees backend running.

The full integration guide is *optional*. These steps are a bit more work and can be done together with the 3DTrees core developer team.

This assumes that you first recursively clone the 3DTrees backend project:

```bash
git clone --recursive git@github.com:3dtrees-earth/3dtrees
cd 3dtrees
git checkout -b <mytool>
```

### Steps from above

A few steps from above need a few adjustments. Basically, you have to add your repo into the 
main repository at the correct location. The tools are all located in the `/tools` folder. 
We suggest to use `tool_<name>` as a naming convention, but that is not strictly necessary.
To add the repository, you created above run the following:

```bash
git submodule add https://github.com/<org_name>/<tool_name> tools/tool_<tool_name>
```

### Steps for Full Integration

The next thing you need to do is create a metadata file about your tool in the local galaxy folder at `/galaxy/tools` (this is the [galaxy repo](https://github.com/3dtrees-earth/galaxy)).
We suggest that you also create a local branch here:

```bash
cd galaxy
git checkout -b mytool
```

Add a new XML metadata file, **you need to use the <toolname> as a filename**.

1. **Create Galaxy Tool XML** (`galaxy/tools/toolname.xml`)

This xml-file specfifies the tool to Galaxy. Please make sure you include:
- `<description>` should be as short as possible! You can provide a longer description in the help section.
- the `<macros>` for correct versioning: The `@TOOL_VERSION` is the version you specify later in the Github versioning process. Make sure they match! The `+galaxy@VERSION_SUFFIX` starts at `0` and increases if the tool itself doesn't change but you make changes to the xml-file leading to a different appearance in the Galaxy GUI.
- `<container>` contains later the link from where the tool will pull the docker image. Make sure to change it later from the local docker image to the registery link. I'll remind you, no worries.
- `<command>` must include `detect_errors="exit_code"` so it doesn't listen to any output in the `stderr` channel but waits for the exit code.
- Provide min and max values for int and float params
- Provide actual tests (check not just for a file as it could be empty but include file size) and work with very small files (<<1MB)
- Include a creator and citation section and please include the correct credits and citations!

```xml
<tool id="3dtrees_tile_merge" name="3Dtrees: Tile and Merge" version="@TOOL_VERSION@+galaxy@VERSION_SUFFIX@" profile="24.2">
    <description>Subsampling, tiling, merging and matching of point clouds</description>
    <macros>
        <token name="@TOOL_VERSION@">1.0.1</token>
        <token name="@VERSION_SUFFIX@">0</token>
    </macros>
    <requirements>
        <container type="docker">ghcr.io/3dtrees-earth/3dtrees_tile_merge:@TOOL_VERSION@</container>
    </requirements>
    <command detect_errors="exit_code"><![CDATA[
        python -u /src/run.py 
        --dataset-path '$input' 
        --output-dir .
        --task '$operation.task'
        #if $operation.task == 'tile':
            --tile-size '$operation.tile_size'
            --overlap '$operation.overlap'
            --tiling-threshold '$operation.tiling_threshold'
            --points-threshold '$operation.points_threshold'
            --subsampling-resolution '$operation.subsampling_resolution'
        #end if
        #if $operation.task == 'merge':
            --buffer '$operation.buffer'
            --min-cluster-size '$operation.min_cluster_size'
            --initial-radius '$operation.initial_radius'
            --max-radius '$operation.max_radius'
            --radius-step '$operation.radius_step'
        #end if
        --number-of-threads \${GALAXY_SLOTS:-4}
    ]]>
    </command>
    <inputs>
        <param name="input" type="data" format="zip,laz" label="Input Point Cloud or ZIP file" help="Input LAS/LAZ point cloud file or ZIP file containing prepared files"/>
        <conditional name="operation">
            <param name="task" type="select" label="Task">
                <option value="tile">Tile</option>
                <option value="merge">Merge</option>
            </param>
            <when value="tile">
                <param argument="--tile-size" type="integer" min="1" max="10000" value="50" label="Tile Size" help="Size of tiles in meters"/>
                <param argument="--overlap" type="integer" min="1" max="10000" value="20" label="Overlap" help="Overlap between tiles in meters"/>
                <param argument="--tiling-threshold" type="float" min="0.1" max="100" value="3" label="Tiling Threshold (GB)" help="File size threshold in GB above which tiling will be applied"/>
                <param argument="--points-threshold" type="integer" min="1" max="100000" value="1000" label="Points Threshold" help="Minimum number of points required per tile - tiles with fewer points will be deleted"/>
                <param argument="--subsampling-resolution" type="integer" min="1" max="100" value="10" label="Subsampling Resolution (cm)" help="Voxel size for subsampling in centimeters (default: 10cm)"/>
            </when>
            <when value="merge">
                <param argument="--buffer" type="float" min="0.1" max="10" value="0.2" label="Buffer Distance (m)" help="Buffer distance for whole-tree assignment (default: 0.2m)"/>
                <param argument="--min-cluster-size" type="integer" min="1" max="10000" value="300" label="Minimum Cluster Size" help="Minimum number of points for a cluster to be considered valid (default: 300)"/>
                <param argument="--initial-radius" type="float" min="0.1" max="10" value="1.0" label="Initial Search Radius (m)" help="Initial radius for point reassignment search (default: 1.0m)"/>
                <param argument="--max-radius" type="float" min="0.1" max="10" value="5.0" label="Maximum Search Radius (m)" help="Maximum radius for point reassignment search (default: 5.0m)"/>
                <param argument="--radius-step" type="float" min="0.1" max="10" value="1.0" label="Radius Step (m)" help="Radius increment step for point reassignment (default: 1.0m)"/>
            </when>
        </conditional>
    </inputs>
    <outputs>
        <data name="output_tile" format="zip" label="Prepared Files" from_work_dir="prepared_files.zip">
            <filter>operation['task'] == "tile"</filter>
        </data>
        <data name="output_merge" format="laz" label="Merged Point Cloud" from_work_dir="final_pc.laz">
            <filter>operation['task'] == "merge"</filter>
        </data>
    </outputs>
    <tests>
        <test expect_num_outputs="1">
            <param name="input" value="mikro.laz"/>
            <conditional name="operation">
                <param name="task" value="tile"/>
                <param name="tile_size" value="50"/>
                <param name="overlap" value="20"/>
                <param name="tiling_threshold" value="3"/>
                <param name="points_threshold" value="1000"/>
                <param name="subsampling_resolution" value="10"/>
            </conditional>
            <output name="output_tile">
                <assert_contents>
                    <has_archive_member path="00_original/input.laz"/>
                    <has_archive_member path="01_subsampled/input_subsampled.laz"/>
                    <has_archive_member path="02_input_SAT/.*\.laz"/>
                </assert_contents>
            </output>
        </test>
        <test expect_num_outputs="1">
            <param name="input" value="processed_files_mikro.zip" />
            <conditional name="operation">
                <param name="task" value="merge"/>
            </conditional>
            <output name="output_merge">
                <assert_contents>
                    <has_size value="200000" delta="100000"/>
                </assert_contents>
            </output>
        </test>
    </tests>
    <help>
        **What it does**
        This tool processes 3D point cloud data for tree segmentation by either:
        - Tiling: Subsampling the input point cloud and creating tiles for processing
        - Merging: Merging processed tiles back into the original point cloud resolution
      ..........
    </help>
    <creator>
        <person name="Kilian Gerberding" email="kilian.gerberding@geosense.uni-freiburg.de" identifier="0009-0002-5001-2571"/>
        <organization name="3Dtrees-Team, University of Freiburg" url="https://github.com/3dTrees-earth"/>
    </creator>
    <citations>
        <citation type="bibtex">
            @misc{3dtrees_tile_merge, title = {3Dtrees Tile and Merge Tool}, author = {3Dtrees Project}, year = {2025}}
        </citation>
    </citations>
</tool>
```

Tip: Provide the necessary metadata, the `parameters.py`, `run.py` and the xml above to an LLM and ask it to write it for you. They do it pretty good.

To help you writing this file, you can install the [Galaxy LSP](https://marketplace.visualstudio.com/items?itemName=davelopez.galaxy-tools) for vscode and cursor.

Finally, you can use `planemo` to validate the XML using:

```bash
# from the project root
planemo lint galaxy/tools/toolname.xml
```

This will yield something like this:
```
Linting tool /Users/mirko/projects/3dtrees/galaxy/tools/overviews.xml
.. CHECK (TestsNoValid): 1 test(s) found.
.. INFO (StdIOAbsenceLegacy): No stdio definition found, tool indicates error conditions with output written to stderr.
.. INFO (OutputsNumber): 3 outputs found.
.. INFO (InputsNum): Found 8 input parameters.
.. CHECK (HelpPresent): Tool contains help section.
.. CHECK (HelpValidRST): Help contains valid reStructuredText.
.. CHECK (ToolIDValid): Tool defines an id [3dtrees_overviews].
.. CHECK (ToolNameValid): Tool defines a name [3D Trees Overview Generator].
.. CHECK (ToolProfileLegacy): Tool targets 16.01 Galaxy profile.
.. CHECK (ToolVersionValid): Tool defines a version [1.0.0].
.. INFO (CommandInfo): Tool contains a command.
.. CHECK (CitationsFound): Found 1 citations.
```

If you messed up the XML (I renamed the `<container>` to `<contner>`), you might get output similar to this:

```
xml.etree.XMLSyntaxError: Opening and ending tag mismatch: contner line 5 and container, line 5, column 69
Could not lint /Users/mirko/projects/3dtrees/galaxy/tools/overviews.xml due to malformed xml.
```

The linter exactly identifies the problem with the XML.

2. **Add build instructions to docker compose**

The main backend repo has already a `docker-compose.yml` at root level. 
You need to contribute the build step for your tool here as well. The docker compose from above needs slight adjustments:

```yaml
services:
    tool-mytool:
        build:
            context: tools/tool_mytool
            dockerfile: Dockerfile
        volumes:
            - ./tools/tool_yourname/in:/in
            - ./tools/tool_yourname/out:/out
        command: ["python", "run.py"]
```

The name patter for the service `tool-<toolname>` **HAS TO MATCH** now, and you also need to adjust the context for docker to build the tool. 
By using the correct names, the `Makefile` can pick up your tool, purge old versions, build your image, link the XML start galaxy, add the tool and link to the container and run the tests in one step.

3. **Use Makefile for Testing**

There are two prepared make shims:
```bash
# Test your tool with Galaxy
make test-tool-<toolname>

# Serve your tool for development
make tool-xml-<toolname>
```

The first one runs the defined tests. It will invoke your tool using a local galaxy instance with the parameters defined in the test and check if the declared result files are actually created. 
**Galaxy only checks file names and Mime types**. The file content is **not** checked.

If your test works, you can also run the second command and open your browser at: `http://127.0.0.1:9090` (note: HTTP, not HTTPS) and you can invoke the tool via the GUI as well.

## Contribute

### 3Dtrees repository
To contribute the tool, **two** PRs are needed now. First, you contribute your `toolname.xml` in the [Galaxy repo](https://github.com/3dtrees-earth/galaxy). 
Second, you contribute the actual tool with a PR in the [3DTrees repo](https://github.com/3DTrees-earth/3dtrees). 

We suggest that you use *+the same name** for both branches and PRs to not get too confused.

### Galaxy toolshed
To make the tool available on [galaxy](https://usegalaxy.eu/) you need to follow the following steps: 

#### Publication of the docker image
To ensure correct versioning of your tool and keep the process as clean as possible, we recommend publishing the docker image using our Github CI workflow. 
Head into the repository of your tool and in the tab "Actions" create a new workflow file (`main.yml`). This will be created in `.github/workflows`. Feel free to copy the following code:
```bash
name: Build and Push Docker Image on Release

on:
  release:
    types: [published]
  workflow_dispatch:

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: 3dtrees-earth/${{ github.event.repository.name }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

This workflow will be triggered once you publish a new version of your tool, extracts the required metadata and builds a versioned Docker image which you can later add to your xml.

If this workflow fails it can be due to the size of your docker image as there is just limited space (~15GB) available. Add the following snippet before the `Build and push Docker image` step. This will take a few more minutes but may resolve your issue.

```bash
      - name: Free space
        run: |
          sudo rm -rf \
            /opt/hostedtoolcache \
            /opt/google/chrome \
            /opt/microsoft/msedge \
            /opt/microsoft/powershell \
            /opt/pipx \
            /usr/local/julia* \
            /usr/local/lib/android \
            /usr/local/lib/node_modules \
            /usr/local/share/chromium \
            /usr/local/share/powershell \
            /usr/share/dotnet \
            /usr/share/swift
```

If you want to include large model weights, you may not be able to provide them directly in the repository. Use [Github LFS](https://docs.github.com/en/repositories/working-with-files/managing-large-files/configuring-git-large-file-storage) to store large files. To include them in the final Docker image you must modify the workflow. 
1. Create an additional release (`eg model_v1`) where you just provide the model file as additional binary file.
2. Edit the workflow `main.yml`:
```bash
env:
  REGISTRY: ghcr.io
  IMAGE_NAME: 3dtrees-earth/${{ github.event.repository.name }}
  MODEL_VERSION: model_v1 #add the release_name
  MODEL_FILE: src/SegmentAnyTree/model_file/PointGroup-PAPER.pt #the path of the model in your docker image

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          lfs: true

      - name: Verify model file
        run: |
          FILE="${{ env.MODEL_FILE }}"
          if [ ! -f "$FILE" ]; then
            echo "::error::Model file not found: $FILE"
            exit 1
          fi

          FILE_SIZE=$(stat -c%s "$FILE")
          echo "Model file size: $FILE_SIZE bytes"

          if [ "$FILE_SIZE" -lt 1000000 ]; then
            echo "::error::Model file appears too small; likely not a valid binary model." #Checks for the file size to make sure it's not the pointer
            exit 1
          fi

          echo "Model file verified."

```
Once this all worked out, modify `container` in the xml accordingly. As it's recommended to work with the macros, make sure the versions match.

#### Add tool specifications to the offical repo
1. Head to the [galaxytools-repo](https://github.com/bgruening/galaxytools/tree/master/tools) and fork it to your profile.
2. Create a new branch with your tool-name.
3. Create a new folder `tools/3Dtrees_tool-name` and add the following items:
   `test-files`: The files you used to test your tool using `make test-tool-your-tool`. Make sure all files are below **1 MB** to keep the size of the rpeo as low as possible.       May feel weird to work with point clouds of a few KB but do it! :)
   `tool-name.xml`: The final tool specification.
   `.shed.yml`: Provides additional information to the toolshed. Create it following the [instructions](https://galaxy-iuc-standards.readthedocs.io/en/latest/best_practices/shed_yml.html#). Please make sure to set `owner: bgruening` and `categories: "Geo Science"` - check out this [example](https://github.com/bgruening/galaxytools/blob/master/tools/3dtrees_tile_merge/.shed.yml). Please keep the name lowercase.
4. Create a pull request and work in the comments of the review process.

#### Request for installation
To have your tool installed, add your tool to [this .yaml file](https://github.com/usegalaxy-eu/usegalaxy-eu-tools/blob/master/bgruening.yaml). Take a look at the other 3dtools for formatting. Your tool will now be updated every Saturday automatically - if you need to have your tool added/updated earlier, reach out to the admins.

#### Request Galaxy resources
If you need access to GPU or need more resources you can request them [here](https://github.com/usegalaxy-eu/infrastructure-playbook/blob/master/files/galaxy/tpv/tools.yml#). Look for your tool and create a PR after the changes. You can adapt the requested ressources to the input file - will provide more information once I've tried that out. The more resources you request the longer the tool will need to actually run.
    

## Extra info

With the `make tool-xml-<toolname>` command, a galaxy instance is started. That also makes the Galaxy API available. 
You can check the [3DTrees API](https://github.com/3dtrees-earth/3dtrees-api) repository, for a full end-to-end integration test for the Overviews tool. This test uses the local infrastructure in the same way as the production system is running.
