# sd-tutorial
Tutorial Apache Flink basado en el playground PyFlink oficial de Apache Flink.

## Elementos
### Kafka
Se usará Kafka para enviar datos de transacciones. Un generador [generate_source_data.py](generator/generate_source_data.py) escribe datos continuamente al topic `payment_msg` de la siguiente manera:
 
`{"createTime": "2020-08-12 06:29:02", "orderId": 1597213797, "payAmount": 28306.44976403719, "payPlatform": 0, "provinceId": 4}`

* `createTime`: The creation time of the transaction. 
* `orderId`: The id of the current transaction.
* `payAmount`: The amount being paid with this transaction.
* `payPlatform`: The platform used to create this payment: pc or mobile.
* `provinceId`: The id of the province for the user. 

### PyFlink

Los datos de las transacciones serán procesados con pyflink con el script [payment_msg_processing.py](payment_msg_proccessing.py).
Este mapea `provinceId` en los registros de input usando su nombre correspondiente con una función definida por el usuario y luego computa la suma los montos de las transacciones para cada provincia. Este trocito de código hace la lógica principal del procesamiento [payment_msg_processing.py](payment_msg_proccessing.py).

```python

t_env.from_path("payment_msg") \ # Get the created Kafka source table named payment_msg
        .select("province_id_to_name(provinceId) as province, payAmount") \ # Select the provinceId and payAmount field and transform the provinceId to province name by a UDF
        .group_by("province") \ # Group the selected fields by province
        .select("province, sum(payAmount) as pay_amount") \ # Sum up payAmount for each province 
        .execute_insert("es_sink") # Write the result into ElaticSearch Sink

```
### ElasticSearch

ElasticSearch se usa para guardar los resultados y hacer querys.

### Kibana

Kibana es un software open para visualizar los datos de ElasticSearch. Se usará el dashboard para visualizar el total de pagos y proporciones de cada provinvia en esta implementación mediante un dashboard.


## Inicio/Ejecución

Abrimos una terminal en la carpeta del proyecto y ejecutamos:
Para armar la imagen de docker:
```bash
docker-compose build
```
Para iniciar:
```bash
docker-compose up-d
```

Chequeamos si se inició:
1. Revisando el Flink Web UI [http://localhost:8081](http://localhost:8081). (Debería haber 1 job disponible y 0 ejecutándose)
2. Revisando el estado de Elasticsearch [http://localhost:9200](http://localhost:9200).
3. Revisando Kibana [http://localhost:5601](http://localhost:5601).

Leemos topic de Kafka en otra terminal para ver si los datos se están enviando correctamente:
```shell script
docker-compose exec kafka kafka-console-consumer.sh --bootstrap-server kafka:9092 --topic payment_msg
```

Iniciamos el "job" de PyFlink:
```shell script
docker-compose exec jobmanager ./bin/flink run -py /opt/pyflink-walkthrough/payment_msg_proccessing.py -d
```

Revisamos nuevamente el Flink Web UI y debería aparecernos el job corriendo.
Luego en Kibana nos vamos al Dashboard y revisamos los gráficos.
### FIN
