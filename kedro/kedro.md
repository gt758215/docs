# **Kedro**

## **Install**
---

```
mkdir kedro-test
cd kedro-test

# optional
python3 -m venv venv
source venv/bin/activate

pip3 install kedro
```


## **Initailize Iris Project by Starter**
---

```
(venv) hua@workst03 ~/kedro-test> python3 -m kedro new --starter=pandas-iris

Project Name
============
Please enter a human readable name for your new project.
Spaces, hyphens, and underscores are allowed.
 [Iris]:iris
...

(venv) hua@workst03:~/kedro-test> ls
iris  venv

(venv) hua@workst03 ~/kedro-test> pip3 install -r iris/src/requirements.txt
```
## **Visualize**
---

```
(venv) hua@workst03 ~/> cd kedro-test/iris

(venv) hua@workst03 ~/kedro-test2> pip3 install kedro-viz

(venv) hua@workst03 ~/k/iris> python3 -m kedro viz --host 10.10.10.101 --port 55121 -a
Do you opt into usage analytics?  [y/N]: N
You have opted out of product usage analytics, so none will be collected.

```
![kedro-viz-jpg](FireShot%20Capture%20019%20-%20Kedro-Viz%20-%20192.168.0.17.jpg)


## **Project Attribute**
---

### **>** <ins> **Data I/O source and format: src** </ins>

* **Attribute:**
```
conf
|-- base
|   |-- catalog.yml
|   |-- logging.yml
|   `-- parameters.yml
|-- local
|   `-- credentials.yml
`-- README.md
```

* **parameters.yml:** Variable, Constant
```
train_fraction: 0.8
random_state: 3
target_column: species
```
* **catalog.yml:** Dataset I/O Parser
```
example_iris_data:
  type: pandas.CSVDataSet
  filepath: data/01_raw/iris.csv
```
* **logging.yml:** logging I/O
```
  info_file_handler:
    class: logging.handlers.RotatingFileHandler
    level: INFO
    formatter: simple
    filename: logs/info.log
    ...
```
### **>** <ins> **Graph build: src** </ins>

* **Attribute:**
```
src
|-- iris
|   |-- __init__.py
|   |-- __main__.py
|   |-- nodes.py
|   |-- pipeline.py
|   |-- pipeline_registry.py
|   |-- README.md
|   `-- settings.py
|-- tests
|   |-- __init__.py
|   |-- test_pipeline.py
|   `-- test_run.py
|-- __init__.py
|-- requirements.txt
`-- setup.py
```
* **node.py:** function -> graph.node
```
def split_data(data: pd.DataFrame, parameters: Dict[str, Any]) -> Tuple[pd.DataFrame, pd.DataFrame, pd.Series, pd.Series]:
    data_train = data.sample(frac=parameters["train_fraction"], random_state=parameters["random_state"])
    data_test = data.drop(data_train.index)

    X_train = data_train.drop(columns=parameters["target_column"])
    X_test = data_test.drop(columns=parameters["target_column"])
    y_train = data_train[parameters["target_column"]]
    y_test = data_test[parameters["target_column"]]

    return X_train, X_test, y_train, y_test
```

* **pipeline.py:** node connection, set node I/O = one of (Node, Variable, Constant, Dataset Input, Label Output)
```
def create_pipeline(**kwargs) -> Pipeline:
    return pipeline(
        [
            node(
                func=split_data,
                inputs=["example_iris_data", "parameters"],
                outputs=["X_train", "X_test", "y_train", "y_test"],
                name="split",
            ),
            ...
        ]
    )
```


## **Demo**
### **>** <ins> **Enable Visualize Tracking** </ins>
---

* vi ~/kedro-test/iris/src/iris/settings.py
```
from kedro_viz.integrations.kedro.sqlite_store import SQLiteStore
from pathlib import Path

SESSION_STORE_CLASS = SQLiteStore
SESSION_STORE_ARGS = {"path": str(Path(__file__).parents[2] / "data")}
```

### **>** <ins> **Add Metrics Output** </ins>
---

* vi ~/kedro-test/iris/conf/base/catalog.yml
```
my_metrics:
  type: tracking.MetricsDataSet
  filepath: data/09_tracking/metrics.json
```

* vi ~/kedro-test/iris/src/iris/nodes.py
```
def report_accuracy(y_pred: pd.Series, y_test: pd.Series):
    accuracy = (y_pred == y_test).sum() / len(y_test)
    logger = logging.getLogger(__name__)
    logger.info("Model has accuracy of %.3f on test data.", accuracy)
    return {"accuracy": accuracy}  ### Modify here
```
* vi ~/kedro-test/iris/src/iris/pipeline.py
```
            ...
            node(
                func=report_accuracy,
                inputs=["y_pred", "y_test"],
                outputs="my_metrics",  ### Modify here
                name="report_accuracy",
            ),
```


### **>** <ins> **Run Pipeline** </ins>
```
(venv) hua@workst03 ~/k/iris> python3 -m kedro run
[11/01/22 13:33:33] INFO     Kedro project iris session.py:343
                    INFO     Loading data from 'example_iris_data' (CSVDataSet)...  data_catalog.py:343
                    INFO     Loading data from 'parameters' (MemoryDataSet)...  data_catalog.py:343
                    INFO     Running node: split: split_data([example_iris_data,parameters]) -> [X_train,X_test,y_train,y_test] node.py:327
                    INFO     Saving data to 'X_train' (MemoryDataSet)...    data_catalog.py:382
                    INFO     Saving data to 'X_test' (MemoryDataSet)... data_catalog.py:382
                    INFO     Saving data to 'y_train' (MemoryDataSet)...    data_catalog.py:382
                    INFO     Saving data to 'y_test' (MemoryDataSet)... data_catalog.py:382
                    INFO     Completed 1 out of 3 tasks sequential_runner.py:85
                    INFO     Loading data from 'X_train' (MemoryDataSet)... data_catalog.py:343
                    INFO     Loading data from 'X_test' (MemoryDataSet)...  data_catalog.py:343
                    INFO     Loading data from 'y_train' (MemoryDataSet)... data_catalog.py:343
                    INFO     Running node: make_predictions: make_predictions([X_train,X_test,y_train]) -> [y_pred] node.py:327
                    INFO     Saving data to 'y_pred' (MemoryDataSet)... data_catalog.py:382
                    INFO     Completed 2 out of 3 tasks sequential_runner.py:85
                    INFO     Loading data from 'y_pred' (MemoryDataSet)...  data_catalog.py:343
                    INFO     Loading data from 'y_test' (MemoryDataSet)...  data_catalog.py:343
                    INFO     Running node: report_accuracy: report_accuracy([y_pred,y_test]) -> [my_metrics]    node.py:327
                    INFO     Model has accuracy of 0.933 on test data.  nodes.py:74
                    INFO     Saving data to 'my_metrics' (MetricsDataSet)...    data_catalog.py:382
                    INFO     Completed 3 out of 3 tasks sequential_runner.py:85
                    INFO     Pipeline execution completed successfully.
```
```
(venv) hua@workst03 ~/k/iris> cat ~/kedro-test/iris/data/09_tracking/metrics.json/2022-11-01T05.39.31.129Z/metrics.json
{
  "accuracy": 0.9333333333333333
}⏎                                     
```
### **>** <ins> **Visualize Tracking** </ins>

![](FireShot%20Capture%20020%20-%20Kedro-Viz%20-%20192.168.0.17.jpg)