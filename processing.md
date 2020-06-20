# Processing
- crie biblioteca para unificar as tarefas comuns, exemplo: conversão de dados, filtros, lógica de negócio
- Crie tipos semâmicos de dados
  - vantagem: tipos semânticos já fazem o feature engineering, pode ajudar no LGPD (ex, ocultar cpf)  
  - ex: nome_cliente, cpf, geo, idade

## Spark
- Com Spark é possível estender o código atual com código legado

### Experimentação Iterativa
- tenha amostras prontas no datalake

#### Exemplo de criação de novos tipos em spark
- Tipos de dados com semântica
```
public class CpfSparkType implements SparkType<String, CpfType> {
@Override
public String serialize(CpfType dataType) {
return dataType.value();
}
@Override
public CpfType deserialize(String sparkData) {
return new CpfType(sparkData);
}
@Override
public DataType sqlType() {
return DataTypes.StringType;
}
}
```

