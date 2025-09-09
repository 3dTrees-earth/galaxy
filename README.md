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
```xml
<tool id="3dtrees_toolname" name="Your Tool Name" version="1.0.0">
    <description>Description of what your tool does</description>
    
    <requirements>
    <container type="docker">3dtrees-tool-toolname:latest</container>
    </requirements>
    
    <command><![CDATA[
        python -u /src/run.py 
        --dataset-path '$input' 
        --output-dir .
        # other params defined in parameters.py
        ]]>
    </command>

    <inputs>
        <param name="input" type="data" format="binary" label="Input Dataset" 
                help="Input file to process"/>
        <!-- Add your specific parameters from parameters.py -->
    </inputs>
    
    <outputs>
        <data name="output" format="your_format" label="Output" from_work_dir="output.file"/>
    </outputs>
    
    <tests>
        <test>
            <param name="input" value="test_input.laz"/>
            <output name="output" file="expected_output.file"/>
        </test>
    </tests>
    
    <help>
        **What it does**
        Detailed description of your tool...
    </help>

    <citations>
        <citation type="bibtex">
            @misc{3dtrees_overviews title = {3D Trees Overview Generator}, author = {3D Trees Project}, year = {2025}
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

If your test works, you can also run the second command and open your browser at: `https://127.0.0.1:9090` and you can invoke the tool via the GUI as well.

## Contribute

To contribute the tool, **two** PRs are needed now. First, you contribute your `toolname.xml` in the [Galaxy repo](https://github.com/3dtrees-earth/galaxy). 
Second, you contribute the actual tool with a PR in the [3DTrees repo](https://github.com/3DTrees-earth/3dtrees). 

We suggest that you use *+the same name** for both branches and PRs to not get too confused.


## Extra info

With the `make tool-xml-<toolname>` command, a galaxy instance is started. That also makes the Galaxy API available. 
You can check the [3DTrees API](https://github.com/3dtrees-earth/3dtrees-api) repository, for a full end-to-end integration test for the Overviews tool. This test uses the local infrastructure in the same way as the production system is running.
