# KotlinHelp

Para criar essa lógica em Kotlin, podemos utilizar o SDK da AWS para interagir com o DynamoDB. Vou demonstrar uma abordagem geral:

1. Primeiro, verificar se existe uma entrada na tabela `tabela1` indicando se as APIs já foram chamadas.
2. Se não houver entrada (ou seja, for a primeira execução), a lógica chama as APIs em sequência e armazena seus estados no DynamoDB.
3. Caso contrário, ele retoma a execução a partir do ponto onde parou na última vez.

Vou exemplificar com o código:

```kotlin
import software.amazon.awssdk.services.dynamodb.DynamoDbClient
import software.amazon.awssdk.services.dynamodb.model.GetItemRequest
import software.amazon.awssdk.services.dynamodb.model.PutItemRequest
import software.amazon.awssdk.services.dynamodb.model.AttributeValue

fun main() {
    val dynamoDbClient = DynamoDbClient.create()
    val tableName = "tabela1"
    val itemKey = "uniqueKey" // Chave para identificar a entrada específica no DynamoDB.

    val currentStatus = getApiStatus(dynamoDbClient, tableName, itemKey)

    // Se o status for nulo, significa que é a primeira execução.
    if (currentStatus == null) {
        println("Primeira execução, chamando APIs em sequência.")
        val updatedStatus = callApisAndStoreStatus(dynamoDbClient, tableName, itemKey, mapOf("api1" to "N", "api2" to "N", "api3" to "N", "api4" to "N"))
        println("Status após chamada das APIs: $updatedStatus")
    } else {
        println("Retomando execução das APIs com base no status armazenado.")
        resumeApisFromLastStatus(dynamoDbClient, tableName, itemKey, currentStatus)
    }
}

fun getApiStatus(dynamoDbClient: DynamoDbClient, tableName: String, key: String): Map<String, String>? {
    val request = GetItemRequest.builder()
        .tableName(tableName)
        .key(mapOf("ItemId" to AttributeValue.builder().s(key).build()))
        .build()

    val response = dynamoDbClient.getItem(request)
    return if (response.hasItem()) {
        response.item().mapValues { it.value.s() }
    } else {
        null
    }
}

fun callApisAndStoreStatus(dynamoDbClient: DynamoDbClient, tableName: String, key: String, status: Map<String, String>): Map<String, String> {
    val newStatus = status.toMutableMap()

    // Simulação das chamadas às APIs e atualização do status
    for (api in listOf("api1", "api2", "api3", "api4")) {
        if (newStatus[api] == "N") {
            println("Chamando $api...")
            // Lógica da chamada da API aqui.
            // Supondo que a chamada foi bem-sucedida, atualizamos o status para "Y".
            newStatus[api] = "Y"
            storeStatusInDynamoDb(dynamoDbClient, tableName, key, newStatus)
        }
    }

    return newStatus
}

fun resumeApisFromLastStatus(dynamoDbClient: DynamoDbClient, tableName: String, key: String, currentStatus: Map<String, String>) {
    val statusToUpdate = currentStatus.toMutableMap()

    // Verifica quais APIs já foram chamadas e continua a partir da primeira que não foi chamada.
    for ((api, status) in statusToUpdate) {
        if (status == "N") {
            println("Retomando chamada da $api...")
            // Lógica da chamada da API aqui.
            statusToUpdate[api] = "Y"
            storeStatusInDynamoDb(dynamoDbClient, tableName, key, statusToUpdate)
        }
    }
}

fun storeStatusInDynamoDb(dynamoDbClient: DynamoDbClient, tableName: String, key: String, status: Map<String, String>) {
    val item = status.mapValues { AttributeValue.builder().s(it.value).build() }.toMutableMap()
    item["ItemId"] = AttributeValue.builder().s(key).build()

    val putItemRequest = PutItemRequest.builder()
        .tableName(tableName)
        .item(item)
        .build()

    dynamoDbClient.putItem(putItemRequest)
    println("Status atualizado no DynamoDB: $status")
}
```

### Explicação
1. **GetItemRequest (`getApiStatus`)**: Verifica se existe uma entrada na tabela `tabela1` para identificar se já houve uma chamada anterior.
2. **Primeira Execução (`callApisAndStoreStatus`)**: Se não houver uma entrada anterior, chamamos as APIs em sequência e armazenamos o status.
3. **Retomada (`resumeApisFromLastStatus`)**: Caso já exista um status armazenado, a execução continua a partir da primeira API que não foi chamada (`status == "N"`).
4. **Persistência (`storeStatusInDynamoDb`)**: Atualiza o status após cada chamada bem-sucedida, garantindo que o estado atual sempre esteja registrado.

Assim, a lógica é capaz de registrar o progresso e retomar de onde parou, caso seja necessário.
