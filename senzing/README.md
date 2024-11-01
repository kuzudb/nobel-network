Run entity resolution on the Nobel Laureate datasets from
<https://github.com/kuzudb/nobel-network>


## Set up the environment

Initialize the Python virtual environment and library dependencies:

```bash
python3 -m venv venv
source venv/bin/activate
python3 -m pip install -U pip wheel setuptools
python3 -m pip install -r requirements.txt
```

Download the [`jq`](https://jqlang.github.io/jq/) tool to convert JSON
to JSONL file format.

Then launch `./venv/bin/jupyter-lab` and from a browser tab load and
run the `data_prep.ipynb` notebook.


## Initialize the Senzing database

Define the Senzing configuration, modifying the Docker platform as
needed, based on your environment:

```bash
export SENZING_ENGINE_CONFIGURATION_JSON='{
  "PIPELINE" : {
    "CONFIGPATH" : "/etc/opt/senzing",
    "RESOURCEPATH" : "/opt/senzing/g2/resources",
    "SUPPORTPATH" : "/opt/senzing/data"
  },
  "SQL" : {
    "CONNECTION": "sqlite3://na:na@/tmp/sqlite/G2C.db"
  }
}'

export DOCKER_DEFAULT_PLATFORM=linux/amd64
```

Then initialize the SQLite database used within Senzing:

```bash
docker run \
    --env SENZING_TOOLS_DATABASE_URL=sqlite3://na:na@/tmp/sqlite/G2C.db \
    --rm \
    --volume /tmp/sqlite:/tmp/sqlite \
    senzing/senzing-tools:0.6.2 init-database
```


## G2 environment

Run within the G2 interactive environment:

```bash
docker run \
    --env SENZING_ENGINE_CONFIGURATION_JSON \
    --rm \
    --volume /tmp/sqlite:/tmp/sqlite \
    --volume /tmp/data:/data \
    -it \
    senzing/senzingapi-tools
```

From the shell command line, download the file utilities within the
container:

```bash
pushd tmp
wget https://raw.githubusercontent.com/senzing-garage/file-loader/refs/heads/main/file-loader.py
wget https://raw.githubusercontent.com/senzing-garage/sz-graph-export/refs/heads/main/sz_graph_export.py
popd
```

Run the `G2ConfigTool.py` interactive CLI to configure Senzing for
this use case:

```
addDataSource LAUREATE
addDataSource SCHOLARS
templateAdd {"feature": "id", "template": "global_id", "behavior": "F1E", "comparison": "exact_comp"}
save
y
exit
```

Load the datasets and run Senzing entity resolution:

```bash
python3 tmp/file-loader.py -f data/laureate.jsonl
python3 tmp/file-loader.py -f data/scholars.jsonl 
```

Finally, export the ER results as a NetworkX graph:
```bash
python3 tmp/sz_graph_export.py -S -o data/demo_
```

The results will now be persisted on the local disk as
`demo_senzing_graph.json`
