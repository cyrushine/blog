```kotlin
fun getMac(): String {
    val mac = runBlocking { 
        withTimeoutOrNull(timeMillis = 2 * 1000) {
            var process: Process? = null
            try {
                val p = Runtime.getRuntime().exec("cat /sys/class/net/wlan0/address ")
                process = p
                val input = LineNumberReader(InputStreamReader(p.inputStream))
                while (true) {
                    val line = input.readLine()
                    if (line != null) {
                        return line.trim()
                    }
                }

            } catch (e: Exception) {
                Log.e(TAG, e.message, e)
            } finally {
                runCatching { process?.destroy() }
            }
            ""
        }
    }
    return mac ?: Const.DEFAULT_MAC
}
```

waitpid 等待别的进程结束，导致 thread 越来越多