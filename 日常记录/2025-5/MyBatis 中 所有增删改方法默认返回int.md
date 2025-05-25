- **MyBatis Mapper 的约定**  
    在 MyBatis 中，所有 `insert`、`update`、`delete` 方法的默认返回值类型都是 `int`，它表示“受影响的行数”（即写库操作实际插入／更新／删除了多少行）。
    
- **Service 层沿用并传递**  
    `DataRainfallCollectionServiceImpl.insertDataRainfallCollection(...)` 接收这个 `int count`，并直接返回：
    `int count = dataRainfallCollectionMapper.insertDataRainfallCollection(data); return count;`
    这样上层（Controller 或其他调用方）就可以通过判断 `count > 0` 来确认写库是否成功，也可以用于批量插入时检查具体插入了几条记录。
    
- **Controller 层统一封装**  
    Controller 端接到的这个 `int` 会被 `toResult(int)` 方法封装成统一的 `RequestResult`，内部通常是这样判断：
    `return count > 0    ? RequestResult.success()    : RequestResult.error("写入失败");`
    因此，使用 `int` 比如 `boolean` 更灵活，不仅能判断成功与否，还能拿到具体的受影响行数。