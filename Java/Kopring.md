```kotlin
@SpringBootApplication
class DemoApplication

fun main(args: Array<String>) {
    runApplication<DemoApplication>(*args)
}
```
```kotlin
import org.springframework.stereotype.Service
import java.util.concurrent.ConcurrentHashMap
import java.util.concurrent.atomic.AtomicLong

data class CreatTodoRequest(
    @field:NotBlank
    val title: String
)

data class TodoResponse(
    val id: Long,
    val title: String,
    val done: Boolean
)

@Service
class TodoService {
    
    private val sequence = AtomicLong(0)
    private val store = ConcurrentHashMap<Long, TodoResponse>()
    
    fun findAll():List<TodoResponse> {
        return store.values.sortedBt { it.id }
    }
    
    fun findById(): TodoResponse {
        return store[id] ?: throw IllegalArgumentException("NOT FOUND")
    }

    fun create(request: CreateTodoRequest): TodoResponse {
        val id = sequence.incrementAndGet()
        val todo = TodoResponse(
            id = id,
            title = request.title,
            done = false
        )
        store[id] = todo
        return todo
    }
}
```