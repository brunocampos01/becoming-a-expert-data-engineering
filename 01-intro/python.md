## Modules
O programa `setup.py` serve para 3 coisas:
- informar os metadados 
- fazer o build do módulo
- fazer a instalação

Para usar um módulo basta importa-lo:
```python
from <module_name> import <function>
```

## Espaço de Memória usado pelo Python
O Python contem muitas funções imbutidas para garantir uma forma rápida e prdutiva de começar a desenvolver.
O Python chama de `namespace` o espaço de memória que reserva para armazenar os nomes de módulos. 
Todo programa Python carrega em seu `namespace` o módulo `__builtins__`, que traz funções como, `range()`, `print()` 