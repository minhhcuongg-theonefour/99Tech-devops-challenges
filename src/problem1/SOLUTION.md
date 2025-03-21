# Problem 1: Too Many Things To Do

## Provided CLI command

```bash
grep -E '"symbol": "TSLA".*"side": "sell"' ./transaction-log.txt | jq -r '.order_id' | xargs -I {} curl -s "https://example.com/api/{}" >> ./output.txt
```

## Explaination

```bash
grep -E '"symbol": "TSLA".*"side": "sell"' ./transaction-log.txt
```
- 1. Search in file ``transaction-log.txt`` and find which line has both ``"symbol": "TSLA"`` and ``"side": "sell"``

```bash
jq -r '.order_id'
```
- 2. From the result got from step (1), using ``jq`` to parse each JSON payload pull out ``"order_id"`` number
```bash
xargs -I {} curl -s "https://example.com/api/{}" >> ./output.txt
```
- 3. Get ID from step 2 and replace to the {} in the URL with each ID using ``xargs`` to extracted ``order_id`` to ``curl`` command, finally append result to output.txt file with ``>> ./output.txt``
