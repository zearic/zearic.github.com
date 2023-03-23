---
title: Create a Spring Boot Rest Controller to accept an uploaded CSV file
tags: "java" "spring boot" "restful" "csv"
---

### Q: Create a Spring Boot Rest Controller to accept an uploaded CSV file

```java
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

@RestController
public class CsvController {

    @PostMapping("/upload")
    public List<String[]> handleFileUpload(@RequestParam("file") MultipartFile file) {
        List<String[]> rows = new ArrayList<>();

        if (file.isEmpty()) {
            return rows;
        }

        try (InputStreamReader reader = new InputStreamReader(file.getInputStream())) {
            CSVReader csvReader = new CSVReaderBuilder(reader).build();
            rows = csvReader.readAll();
        } catch (IOException e) {
            e.printStackTrace();
        }

        return rows;
    }
}
```

This controller uses Spring's `@PostMapping` annotation to map the `/upload` endpoint to the `handleFileUpload` method. The method takes a `MultipartFile` object as a parameter, which represents the uploaded file.

The method checks if the file is empty and returns an empty list if it is. Otherwise, it reads the contents of the file using a `CSVReader` object from the `opencsv` library, which is capable of parsing CSV files. The resulting rows are returned as a list of string arrays.

Note that you'll need to include the `opencsv` dependency in your `pom.xml` file for this code to work:

```xml
<dependency>
    <groupId>com.opencsv</groupId>
    <artifactId>opencsv</artifactId>
    <version>5.5.2</version>
</dependency>
```
