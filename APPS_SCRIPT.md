
# Google Apps Script Code Sahabat Kasir

Copy the code below into your Google Sheet Script Editor (**Extensions > Apps Script**).
**Important:** You must deploy this as a Web App again after updating the code (Deploy > New Deployment).

```javascript
function doGet(e) {
  return handleRequest(e);
}

function doPost(e) {
  return handleRequest(e);
}

function handleRequest(e) {
  var lock = LockService.getScriptLock();
  // Wait up to 30 seconds for other processes to finish.
  lock.tryLock(30000);

  try {
    var doc = SpreadsheetApp.getActiveSpreadsheet();

    // --------------------------------------------
    // 1. READ / RESTORE DATA
    // --------------------------------------------
    
    // Restore All Data
    if (e.parameter && e.parameter.action === 'restore_all') {
      var result = {};
      result.transactions = readSheet(doc, 'Transactions');
      result.inventory = readSheet(doc, 'Inventory');
      result.staff = readSheet(doc, 'Staff');
      result.settings = readSheet(doc, 'Settings');
      result.tasks = readSheet(doc, 'Tasks');
      result.taskTemplates = readSheet(doc, 'TaskTemplates');
      result.storeNeeds = readSheet(doc, 'StoreNeeds');
      result.journal = readSheet(doc, 'Journal');

      return ContentService
        .createTextOutput(JSON.stringify({ status: 'success', data: result }))
        .setMimeType(ContentService.MimeType.JSON);
    }

    // Restore Tasks Only (Realtime Sync)
    if (e.parameter && e.parameter.action === 'restore_tasks') {
        var result = {};
        result.tasks = readSheet(doc, 'Tasks');
        result.taskTemplates = readSheet(doc, 'TaskTemplates');
        return successOutput(result);
    }
    
    // Legacy Read (Transactions only)
    if (e.parameter && e.parameter.action === 'read') {
       var data = readSheet(doc, e.parameter.sheet || 'Transactions');
       return ContentService
        .createTextOutput(JSON.stringify({ status: 'success', data: data }))
        .setMimeType(ContentService.MimeType.JSON);
    }

    // --------------------------------------------
    // 2. WRITE / BACKUP DATA
    // --------------------------------------------
    var postData = JSON.parse(e.postData.contents);

    // Connection Test
    if (postData.id === 'TEST-CONN' || postData.action === 'test') {
       return ContentService
        .createTextOutput(JSON.stringify({ result: 'success' }))
        .setMimeType(ContentService.MimeType.JSON);
    }

    // Realtime: Sync Task (Upsert)
    if (postData.action === 'sync_task') {
        var sheet = getOrCreateSheet(doc, 'Tasks', ['ID', 'Title', 'Category', 'Priority', 'Due', 'Assignees', 'Completed', 'CompletedBy', 'CompletedAt']);
        var data = sheet.getDataRange().getValues();
        var idToSync = String(postData.task.id);
        var rowIndex = -1;
        
        for (var i = 1; i < data.length; i++) {
            if (String(data[i][0]) === idToSync) {
                rowIndex = i + 1;
                break;
            }
        }

        var rowData = [
            postData.task.id,
            postData.task.title,
            postData.task.category,
            postData.task.priority,
            postData.task.due,
            postData.task.assignees,
            postData.task.completed,
            postData.task.completedBy,
            postData.task.completedAt
        ];

        if (rowIndex > 0) {
            sheet.getRange(rowIndex, 1, 1, rowData.length).setValues([rowData]);
            return successOutput({ result: 'updated' });
        } else {
            sheet.appendRow(rowData);
            return successOutput({ result: 'added' });
        }
    }

    // Realtime: Delete Task
    if (postData.action === 'delete_task') {
         var sheet = doc.getSheetByName('Tasks');
         if (sheet) {
             var data = sheet.getDataRange().getValues();
             for (var i = 1; i < data.length; i++) {
                 if (String(data[i][0]) === String(postData.id)) {
                     sheet.deleteRow(i + 1);
                     return successOutput({ result: 'deleted' });
                 }
             }
         }
         return successOutput({ result: 'not_found' });
    }

    // Realtime: Save Journal Entry
    if (postData.action === 'save_journal') {
        var sheet = getOrCreateSheet(doc, 'Journal', ['ID', 'Date', 'Time', 'Content', 'Author']);
        sheet.appendRow([
            postData.row.id,
            postData.row.date,
            postData.row.time,
            postData.row.content,
            postData.row.author
        ]);
        return successOutput({ result: 'added' });
    }

    // Realtime: Save Single Store Need
    if (postData.action === 'save_need') {
        var sheet = getOrCreateSheet(doc, 'StoreNeeds', ['ID', 'Date', 'Items', 'Price', 'AddedBy']);
        var data = sheet.getDataRange().getValues();
        var isDuplicate = false;
        for (var i = 1; i < data.length; i++) {
            if (String(data[i][0]) === String(postData.row.id)) {
                isDuplicate = true;
                break;
            }
        }
        if (!isDuplicate) {
            sheet.appendRow([postData.row.id, postData.row.date, postData.row.items, postData.row.price, postData.row.addedBy]);
            return successOutput({ result: 'added' });
        } else {
            return successOutput({ result: 'duplicate_skipped' });
        }
    }

    // Transaction: Replace / Save (Unified Dynamic Logic)
    if (postData.action === 'replace_rows' || postData.action === 'save_rows') {
        var sheet = getOrCreateSheet(doc, 'Transactions', [
          'No', 'ID', 'Tanggal', 'Jam Pesanan', 'Nama Product', 'Jumlah', 'Harga', 
          'Biaya Tambahan', 'S', 'T', 'M', 'O', 'Discount', 'Total Harga', 
          'Metode Pembayaran', 'Tipe Pesanan', 'Status'
        ]);
        
        // 1. Get Headers
        var headers = sheet.getRange(1, 1, 1, sheet.getLastColumn()).getValues()[0];
        
        // 2. Identify ID Column for replacement
        var idToDelete = postData.action === 'replace_rows' ? String(postData.id) : null;
        if (idToDelete) {
             var data = sheet.getDataRange().getValues();
             var idIndex = -1;
             for(var h=0; h<headers.length; h++) {
                 var cleanH = String(headers[h]).toLowerCase().replace(/[^a-z0-9]/g, '');
                 if(cleanH === 'id' || cleanH === 'orderid') { idIndex = h; break; }
             }
             if (idIndex !== -1) {
                // Delete existing rows for this ID
                for (var i = data.length - 1; i >= 1; i--) {
                    if (String(data[i][idIndex]) === idToDelete) { sheet.deleteRow(i + 1); }
                }
             }
        }

        // 3. Prepare New Rows dynamically based on Headers
        var lastRow = sheet.getLastRow();
        
        // Robust StartNo Calculation: Find max existing No to continue sequence
        var startNo = 1;
        var noColIndex = -1;
        
        // Find 'No' column index
        for(var h=0; h<headers.length; h++) {
            if (String(headers[h]).toLowerCase().replace(/[^a-z0-9]/g, '') === 'no') {
                noColIndex = h + 1;
                break;
            }
        }
        
        if (noColIndex !== -1 && lastRow > 1) {
            try {
                // Get all values in 'No' column
                var colValues = sheet.getRange(2, noColIndex, lastRow - 1, 1).getValues();
                var maxNo = 0;
                for (var r = 0; r < colValues.length; r++) {
                    var val = parseInt(colValues[r][0]);
                    if (!isNaN(val) && val > maxNo) maxNo = val;
                }
                startNo = maxNo + 1;
            } catch(e) {
                startNo = lastRow; // Fallback
            }
        }

        var newRows = [];
        for (var i = 0; i < postData.rows.length; i++) {
            var item = postData.rows[i];
            var rowArray = [];
            
            for (var h = 0; h < headers.length; h++) {
                var header = String(headers[h]);
                var cleanHeader = header.toLowerCase().replace(/[^a-z0-9]/g, '');
                
                // Map common headers to item keys if case differs
                if (cleanHeader === 'no') { 
                    rowArray.push(startNo + i); 
                    continue; 
                }
                
                // Robust key matching
                var val = item[header];
                if (val === undefined) {
                    // Try looking for matching key ignoring case/spaces
                    for (var k in item) {
                        if (k.toLowerCase().replace(/\s/g,'') === header.toLowerCase().replace(/\s/g,'')) {
                            val = item[k];
                            break;
                        }
                    }
                }
                
                if (val === undefined) val = '';
                
                rowArray.push(val);
            }
            newRows.push(rowArray);
        }

        if (newRows.length > 0) {
            sheet.getRange(sheet.getLastRow() + 1, 1, newRows.length, newRows[0].length).setValues(newRows);
        }
        
        return successOutput({ result: postData.action === 'replace_rows' ? 'replaced' : 'saved' });
    }

    // Update Transaction
    if (postData.action === 'update_transaction') {
        var sheet = doc.getSheetByName('Transactions');
        if (!sheet) return successOutput({ result: 'sheet_not_found' });
        var id = String(postData.id);
        var data = sheet.getDataRange().getValues();
        var headerRowIndex = 0;
        var headers = [];
        var idIndex = -1;
        var statusIndex = -1;
        var methodIndex = -1;
        for (var r = 0; r < Math.min(data.length, 5); r++) {
             var tempHeaders = data[r];
             var tempIdIdx = -1;
             for (var h = 0; h < tempHeaders.length; h++) {
                 var val = String(tempHeaders[h]).toLowerCase().replace(/[^a-z0-9]/g, '');
                 if (val === 'id' || val === 'orderid') tempIdIdx = h;
             }
             if (tempIdIdx !== -1) { headerRowIndex = r; headers = tempHeaders; idIndex = tempIdIdx; break; }
        }
        if (idIndex === -1) return successOutput({ result: 'header_not_found' });
        for (var h = 0; h < headers.length; h++) {
            var header = String(headers[h]).toLowerCase().replace(/[^a-z0-9]/g, ''); 
            if (header === 'status' || header === 'orderstatus') statusIndex = h;
            if (header === 'metodepembayaran' || header === 'paymentmethod' || header === 'method') methodIndex = h;
        }
        var updatedCount = 0;
        for (var i = headerRowIndex + 1; i < data.length; i++) {
            if (String(data[i][idIndex]) === id) {
                if (statusIndex !== -1 && postData.status) { sheet.getRange(i + 1, statusIndex + 1).setValue(postData.status); }
                if (methodIndex !== -1 && postData.method) { sheet.getRange(i + 1, methodIndex + 1).setValue(postData.method); }
                updatedCount++;
            }
        }
        return successOutput({ updated: true, rowsUpdated: updatedCount });
    }

    // Overwrites
    if (postData.action === 'backup_inventory') {
        overwriteSheet(doc, 'Inventory', ['ID', 'Name', 'Price', 'Cost', 'Stock', 'Category', 'SKU', 'Unit', 'Image', 'Icon', 'Color', 'IsRaw', 'RecipeJSON'], 
            postData.rows.map(function(r) { return [r.id, r.name, r.price, r.cost, r.stock, r.category, r.sku, r.unit, r.image, r.icon, r.color, r.isRaw, r.recipe]; }));
        return successOutput({ type: 'inventory' });
    }
    if (postData.action === 'backup_staff') {
        overwriteSheet(doc, 'Staff', ['ID', 'Name', 'Role', 'PIN', 'Avatar'], postData.rows.map(function(r) { return [r.id, r.name, r.role, r.pin, r.avatar]; }));
        return successOutput({ type: 'staff' });
    }
    if (postData.action === 'backup_tasks') {
        overwriteSheet(doc, 'Tasks', ['ID', 'Title', 'Category', 'Priority', 'Due', 'Assignees', 'Completed', 'CompletedBy', 'CompletedAt'], 
            postData.rows.map(function(r) { return [r.id, r.title, r.category, r.priority, r.due, r.assignees, r.completed, r.completedBy, r.completedAt]; }));
        return successOutput({ type: 'tasks' });
    }
    if (postData.action === 'backup_needs') {
        overwriteSheet(doc, 'StoreNeeds', ['ID', 'Date', 'Items', 'Price', 'AddedBy'], postData.rows.map(function(r) { return [r.id, r.date, r.items, r.price, r.addedBy]; }));
        return successOutput({ type: 'storeNeeds' });
    }
    if (postData.action === 'backup_journal') {
        overwriteSheet(doc, 'Journal', ['ID', 'Date', 'Time', 'Content', 'Author'], postData.rows.map(function(r) { return [r.id, r.date, r.time, r.content, r.author]; }));
        return successOutput({ type: 'journal' });
    }
    if (postData.action === 'backup_settings') {
        overwriteSheet(doc, 'Settings', ['Key', 'Value'], postData.rows.map(function(r) { return [r.key, r.value]; }));
        return successOutput({ type: 'settings' });
    }
    if (postData.action === 'backup_task_templates') {
        overwriteSheet(doc, 'TaskTemplates', ['Group', 'Title'], postData.rows.map(function(r) { return [r.group, r.title]; }));
        return successOutput({ type: 'taskTemplates' });
    }
    
    return successOutput({ result: 'ignored' });

  } catch (e) {
    return ContentService.createTextOutput(JSON.stringify({ result: 'error', error: e.toString() })).setMimeType(ContentService.MimeType.JSON);
  } finally {
    lock.releaseLock();
  }
}

function successOutput(data) { return ContentService.createTextOutput(JSON.stringify(Object.assign({ result: 'success' }, data))).setMimeType(ContentService.MimeType.JSON); }
function getOrCreateSheet(doc, name, headers) { var sheet = doc.getSheetByName(name); if (!sheet) { sheet = doc.insertSheet(name); sheet.appendRow(headers); } return sheet; }
function readSheet(doc, name) {
    var sheet = doc.getSheetByName(name); if (!sheet) return [];
    var rows = sheet.getDataRange().getValues(); if (rows.length < 2) return [];
    var headers = rows[0]; var data = [];
    for (var i = 1; i < rows.length; i++) {
        var row = rows[i]; var record = {};
        for (var j = 0; j < headers.length; j++) { record[headers[j]] = row[j]; }
        data.push(record);
    }
    return data;
}
function overwriteSheet(doc, name, headers, dataRows) {
    var sheet = doc.getSheetByName(name); if (sheet) { doc.deleteSheet(sheet); }
    sheet = doc.insertSheet(name); sheet.appendRow(headers);
    if (dataRows.length > 0) { sheet.getRange(2, 1, dataRows.length, dataRows[0].length).setValues(dataRows); }
}
```
