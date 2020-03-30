# Begin-Block

每个abci开始块调用，将根据" HistoricalEntries"参数存储并修改历史信息。

## Historical Info Tracking

如果" HistoricalEntries"参数为0，则" BeginBlock"执行空操作。

否则买最新的历史信息将会存储在键`historicalInfoKey|height`下，而所有早于`height - HistoricalEntries`的信息将会删除。在大多数情况下每个块会修改一个条目，但是如果`HistoricalEntries` 被修改的话将会导致一次修改多个条目