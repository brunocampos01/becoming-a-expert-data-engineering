# Leaning About Airflow
Airflow é um workflow manager para execução de task e schedule alem de monitorar todas as execuções.

## Conceitos
Aqui estão os conceitos e termos básicos frequentemente usados ​​no Airflow:

### DAG
Uma DAG é um grupo de tasks que possuem algumas dependências entre si e são executadas de acordo com uma programação. Cada DAG é equivalente a um fluxo de trabalho lógico. 

### Operator
um operator é uma classe Python que atua como um modelo para um determinado tipo de trabalho, por exemplo:
- BashOperator : executa um comando bash
- PythonOperator : execute uma função Python
- PythonVirtualenvOperator : executa uma função em um ambiente virtual que é criado e destruído automaticamente
- PapermillOperator : executar um notebook Jupyter

### Task
Depois que um operator é instanciado com argumentos específicos, ele se torna uma task.

### Instância de Task
Uma instância da task representa uma execução específica de uma tarefa e possui um estado, por exemplo: “running”, “success”, “failed”, “skipped”, “up for retry”, etc.


## Architecture
Possui 4 componentes:
- Data Base: armazena metadados, como por exemplo, estado das tasks. O acesso ao data base acontece usando SQLAlchemy.
- Schedule: é um serviço para definir horários de execução.
- Executor: é um processo de enfileramento de mensagens para garantir a ordem de execução. Essas mensagens são lidas pelo serviço de schedule. É muito útil quando se tem tasks rodando com processos paralelos em máquinas diferentes.
- Workers:  são os processos que realmente executam a lógica das tarefas e são determinados pelo Executor que está sendo usado.

architecture.png

## Running 
O airflow executa os seus processos da seguinte forma:

Etapa 0. Carregue as definições DAG disponíveis do disco (preencha DagBag)

Enquanto o scheduler estiver em execução:
	Etapa 1. O scheduler usa as definições do DAG para 
	        identificar e / ou inicializar qualquer DagRuns no
	        metadados db.
	
	Etapa 2. O scheduler verifica os estados do 
	        TaskInstances associadas a DagRuns ativos, 
		resolve quaisquer dependências entre TaskInstances, 
		identifica TaskInstances que precisam ser executadas, 
		e os adiciona a uma fila de trabalhadores, atualizando o status 
		de TaskInstances recém-enfileiradas para "enfileiradas" no
		base de dados.
	
	Etapa 3. Cada trabalhador disponível extrai um TaskInstance de 
		a fila e começa a executá-la, atualizando o 
	        registro de banco de dados para o TaskInstance de "na fila" 
	        correr".
	
	Etapa 4. Após a conclusão da execução de um TaskInstance, o 
	        trabalhador associado reporta de volta à fila 
	        e atualiza o status da TaskInstance 
	        no banco de dados (por exemplo, "concluído", "falhou", 
	        etc.)
	
	Etapa 5. O planejador atualiza os estados de todos os ativos 
	        DagRuns ("executando", "falhou", "finalizado") de acordo 
	        aos estados de todas as associações associadas concluídas 
	        TaskInstances.
	
	Etapa 6. Repita as etapas 1 a 5

- https://gist.github.com/dustinstansbury/0b43b58542721d20cf6f209436fc68b7#file-scheduler-function-pseudocode

 ## UI
 O grande diferencial do airflow esta na sua interface.

1. Servidor da Web : esse processo executa um aplicativo Flask simples que lê o estado de todas as tarefas do banco de dados de metadados e renderiza esses estados para a interface da Web.
2. Interface da Web : Este componente permite que um usuário do lado do cliente exiba e edite o estado das tarefas no banco de dados de metadados. Devido ao acoplamento entre o Agendador e o banco de dados, a interface do usuário da Web permite que os usuários manipulem o comportamento do agendador.
3. Logs de execução : esses logs são gravados pelos processos de trabalho e armazenados no disco ou em um armazenamento de arquivos remoto (por exemplo, GCS ou S3 ). O servidor da Web acessa os logs e os disponibiliza para a interface da Web.

## Comands

- Executar uma task isoladamente
```bash
airflow test DAG_ID TASK_ID EXECUTION_DATE
```

## DAG
Nenhum processamento de dados real deve ocorrer nos arquivos DAG.

 É essencial manter os arquivos DAG muito leves (como um arquivo de configuração), para que leve menos tempo e recursos para o planejador do Airflow processá-los a cada pulsação .

Sempre use uma estática start_datepara seus DAGs para garantir que as execuções do DAG sejam preenchidas conforme o esperado.

Utilize as variáveis ​​e macros de modelo do Airflow para garantir que suas execuções no DAG sejam independentes uma da outra e do tempo de execução real.


### Default Arguments
Simple example

```
default_args = { 
  'owner': 'xinran.waibel',
  'start_date': datetime(2019, 12, 1),
  'retries': 1,
  'on_failure_callback': slack_failure_msg
}
```

### Task Dependency
```
# Task B depends on Task A and Task C depends on Task B
task_a >> task_b >> task_c

# Task D depends on Task C
task_c.set_downstream(task_d)
```

Uma task que depende de várias task
```
# Task C will run after both Task A and B complete
[task_a, task_b] >> task_c
```


### Simple DAG
Quando criado uma DAG, o scheduler materializa uma lista de execução.

materializ.png

```
default_args = { 
  'owner': 'brunocampos01', 
  'start_date': datetime (2019, 12, 5), 
} 
dag = DAG ('sample_dag', default_args = default_args, schedule_interval = '0 7 * * *')
```
- `start_date` não é necessariamente primeira execução do DAG seria acionada. É só uma referencia para o scheduler saber aonde começa.
- Então, a partir do exemplo, A primeira execução do DAG seria acionada após as 07:00 de 2019-12-06, no final de seu período de programação, em vez de na data de início.

## Complete DAG

```Python
"""
Code that goes along with the Airflow tutorial located at:
https://github.com/apache/incubator-airflow/blob/master/airflow/example_dags/tutorial.py
"""
from airflow import DAG
from airflow.operators.bash_operator import BashOperator
from datetime import datetime, timedelta


default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'start_date': datetime(2015, 6, 1),
    'email': ['airflow@example.com'],
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
    # 'queue': 'bash_queue',
    # 'pool': 'backfill',
    # 'priority_weight': 10,
    # 'end_date': datetime(2016, 1, 1),
}

dag = DAG(
    'tutorial', 
    default_args=dag_test_bash_python, 
    schedule_interval=timedelta(days=1))

# t1, t2 and t3 are examples of tasks 
# created by instantiating operators
t1 = BashOperator(
    task_id='print_date',
    bash_command='date',
    dag=dag)

t2 = BashOperator(
    task_id='sleep',
    bash_command='sleep 5',
    retries=3,
    dag=dag)

templated_command = """
    {% for i in range(5) %}
        echo "{{ ds }}"
        echo "{{ macros.ds_add(ds, 7)}}"
        echo "{{ params.my_param }}"
    {% endfor %}
"""

t3 = BashOperator(
    task_id='templated',
    bash_command=templated_command,
    params={'my_param': 'Parameter I passed in'},
    dag=dag)

t2.set_upstream(t1)
t3.set_upstream(t1)
```

## Start Service
```bash
# airflow needs a home, ~/airflow is the default,
# but you can lay foundation somewhere else if you prefer
# (optional)
export AIRFLOW_HOME=~/airflow

# install from pypi using pip
pip install apache-airflow

# initialize the database
airflow initdb

# start the web server, default port is 8080
airflow webserver -p 8080

# start the scheduler
airflow scheduler

# visit localhost:8080 in the browser and enable the example dag in the home page
```

## Validations and Tests
## DAG
- First check that DAG file contains valid Python code
```bash
python3 airflow/dags/airflow_tutorial.py
```

- Verifique se a DAG já consta na lista de DAG 
```bash
airflow list_dags

# -------------------------------------------------------------------
# DAGS
# -------------------------------------------------------------------
# dag_test_bash_python
# example_bash_operator
# example_branch_dop_operator_v3
# example_branch_operator
# example_http_operator
# example_passing_params_via_test_command
# example_pig_operator
# example_python_operator
# example_short_circuit_operator
# example_skip_dag
# example_subdag_operator
# example_subdag_operator.section-1
# example_subdag_operator.section-2
# example_trigger_controller_dag
# example_trigger_target_dag
# example_xcom
# latest_only
# latest_only_with_trigger
# test_utils
# tutorial
```

- Show the hierarchy of task in DAG
```bash
# prints the hierarchy of tasks in the tutorial DAG
airflow list_tasks tutorial --tree
```

### Tasks
Test each task using `airflow test`
```bash
airflow test example_bash_operator runme_0 2015-01-01
```










## catchup
O catchup é um atributo de uma DAG para garantir que orquestração dos jobs nos horários corretos

```
dag.catchup = True
```

Quando o catchup está ativada e cada execução do DAG pode ser executada manualmente a qualquer momento, é importante garantir que os DAGs sejam idempotentes e que cada execução do DAG seja independente uma da outra e da data real da execução.

#### Reference
- https://towardsdatascience.com/getting-started-with-apache-airflow-df1aa77d7b1b