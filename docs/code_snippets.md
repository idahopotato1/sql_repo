### Collections of node snippets 

#### Tabulator hide column snowlow - solution is to disable table redraw
```python 
table.blockRedraw();
  for (const col of table.getColumns()) {
    if (!firstCol) {
      col.toggle();
    }
    firstCol = false;
  }
table.restoreRedraw();
```