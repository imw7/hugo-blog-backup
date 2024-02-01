+++
title='Excel VBA 常用代码'
tags=["VBA"]
categories=["VBA"]
date="2023-04-25T18:07:12+08:00"
toc=true
draft=false

+++

本文记录了 `Excel VBA` 的常用工具，方便取用。<!--more-->

### 批量创建工作表

```visual basic
Sub NewSht()
    Dim shtActive As Worksheet, sht As Worksheet
    Dim i As Long, strShtName As String
    On Error Resume Next '当代码出错时继续运行
    Set shtActive = ActiveSheet
    For i = 2 To shtActive.Cells(Rows.Count, 1).End(xlUp).Row
    '单元格A1是标题，跳过，从第2行开始遍历工作表名称
        strShtName = shtActive.Cells(i, 1).Value
        '工作表名强制转换为字符串类型
        Set sht = Sheets(strShtName)
        '当工作簿不存在工作表Sheets(strShtName)时，这句代码会出错，然后……
        If Err Then
        '如果代码出错，说明不存在工作表Sheets(t)，则新建工作表
            Worksheets.Add , Sheets(Sheets.Count)
            '新建一个工作表，位置放在所有已存在工作表的后面
            ActiveSheet.Name = strShtName
            '新建的工作表必然是活动工作表，为之命名
            Err.Clear
            '清除错误状态
        End If
    Next
    shtActive.Activate
    '重新激活原工作表
End Sub
```

### 删除全部工作表

```visual basic
Sub DelShet() '删除所有工作表
    Dim sht As Worksheet
    Application.ScreenUpdating = False '关屏幕刷新
    Application.DisplayAlerts = False '关警告信息
    On Error Resume Next
    For Each sht In Worksheets
        sht.Delete '遍历工作表删除
    Next
    Application.ScreenUpdating = True
    Application.DisplayAlerts = True
End Sub
```

### 提取工作表名字

```visual basic
Sub GetShtByVba()
    Dim sht As Worksheet, k As Long
    Application.ScreenUpdating = False
    k = 1
    Range("a:b").Clear '清空数据
    Range("a:a").NumberFormat = "@" '设置文本格式
    For Each sht In Worksheets '遍历工作表取表名
        k = k + 1
        Cells(k, 1) = sht.Name
    Next
    Range("a1:b1") = Array("工作表名", "是否删除")
    Application.ScreenUpdating = True
End Sub
```

### 删除指定工作表

```visual basic
Sub DelShtByVba()
    Dim sht As Worksheet, i As Long, r
    Application.ScreenUpdating = False
    Application.DisplayAlerts = False
    On Error Resume Next
    r = Range("a1").CurrentRegion '数据装入数组r
    For i = 2 To UBound(r) '遍历并删除工作表
        If r(i, 2) = "删除" Then Worksheets(CStr(r(i, 1))).Delete
    Next
    Application.ScreenUpdating = True
    Application.DisplayAlerts = True
End Sub
```

### 生成带超链接的工作表目录

```visual basic
Sub ml()
    Dim sht As Worksheet, i&, strShtName$
    Columns(1).ClearContents '清空A列数据
    Cells(1, 1) = "目录" '第一个单元格写入标题"目录"
    i = 1  '将i的初值设置为1.
    For Each sht In Worksheets  '循环当前工作簿的每个工作表
        strShtName = sht.Name
        If strShtName <> ActiveSheet.Name Then
       '如果sht的名称不是当前工作表的名称则开始在当前工作表建立超链接
            i = i + 1 '累加工作表数量
           ActiveSheet.Hyperlinks.Add anchor:=Cells(i, 1), Address:="", _
            SubAddress:="'" & strShtName & "'!a1", TextToDisplay:=strShtName
           '建超链接
        End If
    Next
End Sub
```

### 在各个分表创建返回总表的命令按钮

```visual basic
Dim strShtName As String
Sub Mybutton()
    Dim sht As Worksheet, btn As Button
    On Error Resume Next
    For Each sht In Worksheets
        With sht
            If .Name <> strShtName Then
                .Shapes(strShtName).Delete
                '删除原有的名称为shtn的按钮，避免重复创建
                Set btn = .Buttons.Add(0, 0, 60, 30)'使用add方法在工作表中添加一个按钮控件，add方法语法如下:表达式.Add(left,right,width,height)
                '新建按钮，释义见小贴士
                With btn
                    .Name = strShtName
                    '命令按钮命名
                    .Characters.Text = "返回总表"
                    '按钮的文本内容
                    .OnAction = "LinkTable"
                    '指定按钮控件所执行的宏命令
                End With
            End If
        End With
    Next
    Set btn = Nothing
End Sub

Sub LinkTable()
    strShtName = "总表"'指定了返回总表的名字，可以根据实际需要修改为目标表的名称，比如“目录”。
    '设置变量strShtName为总表的名称，可以根据实际总表的名称做修改
    Worksheets(strShtName).Activate
    [a1].Select
End Sub
```

### 批量提取工作表的名字【推荐】

```visual basic
Sub GetShtName()
    Dim sht As Worksheet, i As Long
    i = 1 'i初始值为1
    With Columns(1)
        .ClearContents '清除A列内容
        .NumberFormat = "@" '设置单元格格式为文本
    End With
    Cells(1, 1) = "工作表名称目录"
    For Each sht In Worksheets '遍历工作表
        i = i + 1
        Cells(i, 1) = sht.Name '在A列记录工作表名称
    Next
End Sub
```

### 批量修改工作表的名字

```visual basic
Sub ReNameSht()
    Dim strShtName$, sht As Worksheet, i&
    On Error Resume Next '当程序运行中出现错误时，继续运行
    For i = 2 To Cells(Rows.Count, 1).End(xlup).Row '遍历当前表格A列的数据
        strShtName = Cells(i, 1).Value '将表格A列的值，赋予变量strShtName
        Worksheets(strShtName).Name = Cells(i, 2).Value '工作表重命名
    Next
End Sub
```

### 批量取消工作表的隐藏

```visual basic
Sub unShtVisible()
    Dim sht As Worksheet
    For Each sht In Worksheets '遍历工作表，设置可见
        sht.Visible = xlSheetVisible
    Next
End Sub
```

### 一键汇总各分表数据成总表【不保留分表格式】

```visual basic
Sub CollectData()
    Dim Sht As Worksheet, rng As Range, k&, n&
    Application.ScreenUpdating = False
    '取消屏幕更新
    n = Val(InputBox("请输入标题的行数", "提醒"))
    If n < 0 Then MsgBox "标题行数不能为负数。", 64, "提示": Exit Sub
    '取得用户输入的标题行数，如果为负数，退出程序
    Cells.ClearContents
    '清空当前表数据
    For Each Sht In Worksheets
    '遍历工作表
        If Sht.Name <> ActiveSheet.Name Then
        '如果工作表名称不等于当前表名则进行汇总动作……
            Set rng = Sht.UsedRange
            '定义rng为表格已用区域
            k = k + 1
            '累计K值
            If k = 1 Then
            '如果是首个表格，则K为1，则把标题行一起复制到汇总表
                rng.Copy
                [a1].PasteSpecial Paste:=xlPasteValues '仅粘贴数值
            Else
                '否则，扣除标题行后再复制黏贴到总表，只黏贴数值
                rng.Offset(n).Copy
                Cells(ActiveSheet.UsedRange.Rows.Count + 1, 1).PasteSpecial Paste:=xlPasteValues
            End If
        End If
    Next
    [a1].Activate
    Application.ScreenUpdating = True '恢复屏幕刷新
End Sub
```

### 汇总分表成总表【保留分表格式】

```visual basic
Sub CollectDataFromShtFormat()
    Dim sht As Worksheet, rng As Range, k As Long, nTitleCount As Long
    On Error Resume Next
    nTitleCount = Val(InputBox("请输入标题的行数", "提醒", 1))
    If nTitleCount < 0 Then MsgBox "标题行数不能为负数。", 64, "提示": Exit Sub
    Application.ScreenUpdating = False
    Cells.ClearContents '清空当前表数据
    For Each sht In Worksheets '遍历工作表
        If sht.Name <> ActiveSheet.Name Then
        '如果工作表名称不等于当前表名则进行汇总动作……
            Set rng = sht.UsedRange
            k = k + 1 '累计K值
            If k = 1 Then '如果是首个表格，则K为1，则把标题行一起复制到汇总表
                sht.Cells.Copy: Range("a1").PasteSpecial Paste:=xlPasteFormats '只粘贴格式
                rng.Copy: Range("a1").PasteSpecial Paste:=xlPasteValues '只粘贴数值
            Else '否则，扣除标题行后再复制黏贴到总表，只黏贴数值
                rng.Offset(nTitleCount).Copy
                With Cells(ActiveSheet.UsedRange.Rows.Count + 1, 1)
                    .PasteSpecial Paste:=xlPasteFormats '粘贴格式
                    .PasteSpecial Paste:=xlPasteValues '粘贴数值
                End With
            End If
        End If
    Next
    Range("a1").Activate
    Application.ScreenUpdating = True '恢复屏幕刷新
    MsgBox "汇总OK，一共汇总了：" & k & "张工作表"
End Sub
```

### 工作表排序

#### 提取所有工作表名

```visual basic
Sub GetShtName()
    Dim k As Long, sht As Worksheet
    Application.ScreenUpdating = False
    With Columns(1)
        .ClearContents '清空A列原有数据
        .NumberFormat = "@" '设置单元格格式为文本
    End With
    Cells(1, 1) = "目录"
    k = 1
    For Each sht In ThisWorkbook.Worksheets '遍历工作表
        If sht.Name <> ActiveSheet.Name Then '如果sht不等于当前工作表名称
            k = k + 1 '累加工作表个数
            Cells(k, 1) = sht.Name '工作表名称写入A列
        End If
    Next
    Application.ScreenUpdating = True
End Sub
```

#### 按照指定顺序排列工作表

```visual basic
Sub SortSht()
    Dim shtActive As Worksheet, i As Long
    Dim arr, strShtName As String
    On Error Resume Next
    Application.ScreenUpdating = False
    Set shtActive = ActiveSheet '当前表赋值变量shtactive
    arr = Range("a1:a" & Cells(Rows.Count, 1).End(xlUp).Row)
    'A列数据装入数组arr
    For i = 2 To UBound(arr) '遍历数组arr
        strShtName = arr(i, 1)
        Worksheets(strShtName).Move after:=Worksheets(i - 1)
        '指定工作表按顺序排放
    Next
    shtActive.Select '回到操作表
    Application.ScreenUpdating = True
End Sub
```

### 批量工作表加密

```visual basic
Sub ProtectSht()
    Dim strAds As String, sht As Worksheet
    Dim strKey As String, strTemp As String
    Dim rng As Range, strMsg As String
    Dim strNoShtName As String, strYesShtName As String
    On Error Resume Next
    strAds = InputBox("请输入单元格保存范围，例如A1:B10." & vbCr _
                                & "可以设置不连续单元格，中间请以逗号分隔。比如A1:B10,D2:D8" & vbCr _
                                & "如果需要全表保护，可以直接确定。", Default:="全表保护")
    If StrPtr(strAds) = False Then Exit Sub
    If strAds = "全表保护" Then strAds = Cells.Address
    Set rng = Range(strAds) '测试输入的单元格区域是否有效
    If Err Then MsgBox "你输入的单元格区域地址不是正确的格式，请重新操作。": Exit Sub
    strKey = InputBox("请输入保护密码。") '第一次输入密码
    If StrPtr(strKey) = False Then Exit Sub
    strTemp = InputBox("请再次输入保护密码。") '第二次输入密码
    If StrPtr(strKey) = False Then Exit Sub
    If strKey <> strTemp Then MsgBox "你两次输入的密码不一致，系统退出，请重新操作。": Exit Sub
    For Each sht In Worksheets '遍历工作表加密保护
        With sht
            If .ProtectContents = False Then '如果工作表未保护
                .Cells.Locked = False '全部单元格区域取消锁定
                .Range(strAds).Locked = True '需要保护的区域锁定
                .Protect strKey, True, True, True '保护工作表，只允许编辑非锁定区域
                strYesShtName = strYesShtName & "," & .Name '保护成功的工作表名称
            Else
                strNoShtName = strNoShtName & "," & .Name '自身已有保护功能的工作表
            End If
        End With
    Next
    If strYesShtName <> "" Then strMsg = "工作表：" & Mid(strYesShtName, 2) & "的" & strAds & "区域保护完成"
    If strNoShtName <> "" Then strMsg = strMsg & vbCrLf & "以下工作表自身已有保护，无法再次保护：" & Mid(strNoShtName, 2)
    MsgBox (strMsg)
End Sub
```

### 批量工作表解密

```visual basic
Sub UnProtct()
    MsgBox "破解提示：当要求输入密码时请点击取消！”"
    Application.DisplayAlerts = False
    On Error Resume Next
    Dim sht As Worksheet
    For Each sht In Worksheets
        With sht
            .Protect DrawingObjects:=True, Contents:=True, Scenarios:=True, AllowFiltering:=True, AllowUsingPivotTables:=True
            .Protect DrawingObjects:=False, Contents:=True, Scenarios:=False, AllowFiltering:=True, AllowUsingPivotTables:=True
            .Protect DrawingObjects:=True, Contents:=True, Scenarios:=False, AllowFiltering:=True, AllowUsingPivotTables:=True
            .Protect DrawingObjects:=False, Contents:=True, Scenarios:=True, AllowFiltering:=True, AllowUsingPivotTables:=True
            .Unprotect
        End With
    Next
    MsgBox "ok"
End Sub
```

### 按任意列拆分多个表

#### 方法一

```visual basic
Sub SplitShts()
    Dim d As Object, sht As Worksheet
    Dim aData, aResult, aTemp, aKeys, i&, j&, k&, x&
    Dim rngData As Range, rngGist As Range
    Dim lngTitleCount&, lngGistCol&, lngColCount&
    Dim rngFormat As Range, aRef, strYesOrNo As String
    Dim strKey As String, strTemp As String
    On Error Resume Next '忽略错误，程序继续运行
    Set d = CreateObject("scripting.dictionary")
    Set rngGist = Application.InputBox("请框选拆分依据列！只能选择单列单元格区域！", Title:="提示", Type:=8)
    '用户选择的拆分依据列
    lngGistCol = rngGist.Column
    '拆分依据列的列标
    lngTitleCount = Val(Application.InputBox("请输入总表标题行的行数？", Default:=1))
    '用户设置总表的标题行数
    If lngTitleCount < 0 Then MsgBox "标题行数不能为负数，程序退出。": Exit Sub
    strYesOrNo = MsgBox("是否需要在分表保留总表格式？", vbYesNo)
    Set rngData = rngGist.Parent.UsedRange
    '总表的数据区域
    Set rngFormat = rngGist.Parent.Cells
    '总表的单元格区域用于粘贴总表格式
    aData = rngData.Value '数据源装入数组
    lngGistCol = lngGistCol - rngData.Column + 1
    '计算依据列在数组中的位置
    lngColCount = UBound(aData, 2)
    '数据源的列数
    Application.ScreenUpdating = False
    Application.DisplayAlerts = False
    ReDim aRef(1 To UBound(aData))
    For i = 1 To UBound(aData) '处理依据列的异常值，空白/错误值/整行空白等
        If IsError(aData(i, lngGistCol)) Then
            aRef(i) = "错误值"
        ElseIf aData(i, lngGistCol) = "" Then
            strTemp = "" '判断是否整行数据为空
            For j = 1 To lngColCount
                strTemp = strTemp & aData(i, j)
            Next
            If strTemp = "" Then '如果整行为空
                aRef(i) = "整行空白"
            Else
                aRef(i) = "空白单元格"
            End If
        Else
            strKey = aData(i, lngGistCol)
            aRef(i) = strKey
        End If
    Next
    For i = lngTitleCount + 1 To UBound(aData)
        strKey = aRef(i)
        If strKey <> "整行空白" Then
            If Not d.exists(strKey) Then
            '字典中不存在关键字时则遍历建表
                d(strKey) = ""
                ReDim aResult(1 To UBound(aData), 1 To lngColCount) '声明一个结果数组
                k = 0
                For x = lngTitleCount + 1 To UBound(aData) '遍历数据源
                    strTemp = aRef(x)
                    If strTemp = strKey Then '如果记录符合条件，则装入结果数组
                        k = k + 1
                        For j = 1 To lngColCount
                            aResult(k, j) = aData(x, j)
                        Next
                    End If
                Next
                For Each sht In ActiveWorkbook.Worksheets '删除旧表
                    If sht.Name = strKey Then sht.Delete
                Next
                With Worksheets.Add(, Sheets(Sheets.Count))
                '新建一个工作表
                    .Name = strKey
                    .Range("a1").Resize(UBound(aData), lngColCount).NumberFormat = "@"
                    '设置单元格为文本格式
                    If lngTitleCount > 0 Then .Range("a1").Resize(lngTitleCount, lngColCount) = aData
                    '标题行
                    .Range("a1").Offset(lngTitleCount, 0).Resize(k, lngColCount) = aResult
                    '写入数据
                    If strYesOrNo = vbYes Then '如果用户选择保留总表格式
                        rngFormat.Copy
                        .Range("a1").PasteSpecial Paste:=xlPasteFormats, Operation:=xlNone, SkipBlanks:=False, Transpose:=False
                         '复制粘贴总表的格式
                        .Range("a1").Offset(lngTitleCount + k, 0).Resize(UBound(aData) - k - lngTitleCount, 1).EntireRow.Delete
                        '删除多余的格式单元格
                    End If
                    .Range("a1").Select
                End With
            End If
        End If
    Next
    rngData.Parent.Activate '回到总表
    Application.ScreenUpdating = True
    Application.DisplayAlerts = True
    Set d = Nothing
    Set rngData = Nothing
    Set rngGist = Nothing
    Set rngFormat = Nothing
    Erase aData: Erase aResult
    MsgBox "数据拆分完成！"
End Sub
```

#### 方法二

```visual basic
Private Sub Worksheet_Change(ByVal Target As Range)
   ActiveWorkbook.RefreshAll
End Sub
```

### 批量将工作表转换为独立的工作簿

```visual basic
Sub EachShtToWorkbook()
    Dim sht As Worksheet, strPath As String
    With Application.FileDialog(msoFileDialogFolderPicker)
   '选择保存工作薄的文件路径
        If .Show Then strPath = .SelectedItems(1) Else Exit Sub
        '读取选择的文件路径,如果用户未选取路径则退出程序
    End With
    If Right(strPath, 1) <> "\" Then strPath = strPath & "\"
    Application.DisplayAlerts = False
    '取消显示系统警告和消息，避免重名工作簿无法保存。当有重名工作簿时，会直接覆盖保存。
    Application.ScreenUpdating = False '取消屏幕刷新
    For Each sht In Worksheets '遍历工作表
        sht.Copy '复制工作表，工作表单纯复制后，会成为活动工作薄
        With ActiveWorkbook
            .SaveAs strPath & sht.Name, xlWorkbookDefault
            '保存活动工作薄到指定路径下，以当前系统默认文件格式
            .Close True '关闭工作薄并保存
        End With
    Next
    MsgBox "处理完成。", , "提醒"
    Application.ScreenUpdating = True '恢复屏幕刷新
    Application.DisplayAlerts = True '恢复显示系统警告和消息
End Sub
```

### 将总表按任意列拆分成多个工作簿

```visual basic
Sub SplitShts()
    Dim d As Object, sht As Worksheet
    Dim aData, aResult, aTemp, aKeys, i&, j&, k&, x&
    Dim rngData As Range, rngGist As Range, ws As Workbook
    Dim lngTitleCount&, lngGistCol&, lngColCount&
    Dim rngFormat As Range, aRef, strYesOrNo As String
    Dim strKey As String, strTemp As String, strPath As String
    On Error Resume Next '忽略错误，程序继续运行
    Set d = CreateObject("scripting.dictionary")
    With Application.FileDialog(msoFileDialogFolderPicker)
    '用户选择保存工作簿的路径
        If .Show Then strPath = .SelectedItems(1) Else Exit Sub
    End With
    If Right(strPath, 1) <> "\" Then strPath = strPath & "\"
    Set rngGist = Application.InputBox("请框选拆分依据列！只能选择单列单元格区域！", Title:="提示", Type:=8)
    '用户选择的拆分依据列
    If rngGist Is Nothing Then Exit Sub
    lngGistCol = rngGist.Column '拆分依据列的列标
    lngTitleCount = Val(Application.InputBox("请输入总表标题行的行数？", Default:=1))
    '用户设置总表的标题行数
    If lngTitleCount < 0 Then MsgBox "标题行数不能为负数，程序退出。": Exit Sub
    strYesOrNo = MsgBox("是否需要在分表保留总表格式？", vbYesNo)
    Set rngData = rngGist.Parent.UsedRange
    '总表的数据区域
    Set rngFormat = rngGist.Parent.Cells
    '总表的单元格区域用于粘贴总表格式
    aData = rngData.Value '数据源装入数组
    lngGistCol = lngGistCol - rngData.Column + 1
    '计算依据列在数组中的位置
    lngColCount = UBound(aData, 2)
    '数据源的列数
    Application.ScreenUpdating = False
    Application.DisplayAlerts = False
    ReDim aRef(1 To UBound(aData))
    For i = 1 To UBound(aData) '处理依据列的异常值，空白/错误值/整行空白等
        If IsError(aData(i, lngGistCol)) Then
            aRef(i) = "错误值"
        ElseIf aData(i, lngGistCol) = "" Then
            strTemp = "" '判断是否整行数据为空
            For j = 1 To lngColCount
                strTemp = strTemp & aData(i, j)
            Next
            If strTemp = "" Then '如果整行为空
                aRef(i) = "整行空白"
            Else
                aRef(i) = "空白单元格"
            End If
        Else
            strKey = aData(i, lngGistCol)
            aRef(i) = strKey
        End If
    Next
    For i = lngTitleCount + 1 To UBound(aData)
        strKey = aRef(i)
        If strKey <> "整行空白" Then
            If Not d.exists(strKey) Then
            '字典中不存在关键字时则遍历建表
                d(strKey) = ""
                ReDim aResult(1 To UBound(aData), 1 To lngColCount) '声明一个结果数组
                k = 0
                For x = lngTitleCount + 1 To UBound(aData) '遍历数据源
                    strTemp = aRef(x)
                    If strTemp = strKey Then '如果记录符合条件，则装入结果数组
                        k = k + 1
                        For j = 1 To lngColCount
                            aResult(k, j) = aData(x, j)
                        Next
                    End If
                Next
                Set ws = Workbooks.Add
                With ws.Sheets(1)
                '新建一个工作簿
                    .Range("a1").Resize(UBound(aData), lngColCount).NumberFormat = "@"
                    '设置单元格为文本格式
                    If lngTitleCount > 0 Then .Range("a1").Resize(lngTitleCount, lngColCount) = aData
                    '标题行
                    .Range("a1").Offset(lngTitleCount, 0).Resize(k, lngColCount) = aResult
                    '写入数据
                    If strYesOrNo = vbYes Then '如果用户选择保留总表格式
                        rngFormat.Copy
                        .Range("a1").PasteSpecial Paste:=xlPasteFormats, Operation:=xlNone, SkipBlanks:=False, Transpose:=False
                         '复制粘贴总表的格式
                        .Range("a1").Offset(lngTitleCount + k, 0).Resize(UBound(aData) - k - lngTitleCount, 1).EntireRow.Delete
                        '删除多余的格式单元格
                    End If
                    .Range("a1").Select
                End With
                ws.SaveAs strPath & strKey, xlWorkbookDefault
                ws.Close False
            End If
        End If
    Next
    Application.ScreenUpdating = True
    Application.DisplayAlerts = True
    Set d = Nothing
    Set rngData = Nothing
    Set rngGist = Nothing
    Set rngFormat = Nothing
    Erase aData: Erase aResult
    MsgBox "数据拆分完成！"
End Sub
```

### 选中行或列会填充颜色

```visual basic
Private Sub Workbook_SheetSelectionChange(ByVal Sh As Object, ByVal Target As Range)
    Application.ScreenUpdating = False
    Cells.Interior.ColorIndex = -4142 '取消单元格原有填充色，但不包含条件格式产生的颜色。
    Rows(Target.Row).Interior.ColorIndex = 33 '活动单元格整行填充颜色
    Columns(Target.Column).Interior.ColorIndex = 33 '活动单元格整列填充颜色
    Application.ScreenUpdating = True
End Sub
```

### 按指定名称批量创建工作簿

```visual basic
Sub CreateFiles()
    Dim strPath As String, strFileName As String
    Dim i As Long, r
    On Error Resume Next
    With Application.FileDialog(msoFileDialogFolderPicker)
        '用户选择文件夹路径
        If .Show Then strPath = .SelectedItems(1) Else Exit Sub
        '如果用户为选择文件夹则退出程序
    End With
    If Right(strPath, 1) <> "\" Then strPath = strPath & "\"
    Application.ScreenUpdating = False '取消屏幕刷新
    Application.DisplayAlerts = False '取消警告提示，当有重名工作簿时直接覆盖
    r = Range("a1:a" & Cells(Rows.Count, 1).End(xlUp).Row) '数据装入数组r
    For i = 2 To UBound(r) '标题不要，因此从第2个元素开始遍历数组r
        With Workbooks.Add '新建工作簿
            .SaveAs strPath & r(i, 1), xlWorkbookDefault
            '以指定名称、默认文件类型保存工作簿
            .Close True '关闭工作簿
        End With
    Next
    Application.ScreenUpdating = True
    Application.DisplayAlerts = True
    MsgBox "创建完成。"
End Sub
```

### 按指定条件批量删除工作簿

#### 提取工作簿

```visual basic
Sub GetFiles()
    Dim strPath As String, strFileName As String, k As Long
    With Application.FileDialog(msoFileDialogFolderPicker)
        If .Show Then strPath = .SelectedItems(1) Else: Exit Sub
        '获取用户选择的文件夹的路径，如果未选取，则退出程序
    End With
    If Right(strPath, 1) <> "\" Then strPath = strPath & "\"
    Application.ScreenUpdating = False
    Range("a:b").Clear: k = 1
    '清除A:B列的所有
    Cells(1, 1) = "旧文件名": Cells(1, 2) = "是否删除"
    strFileName = Dir(strPath & "*.xls*")
    Do While strFileName <> ""
        k = k + 1
        Cells(k, 1) = strPath & strFileName
        strFileName = Dir
    Loop
    Application.DisplayAlerts = True
End Sub
```

#### 删除指定工作簿

```visual basic
Sub DeleteFile()
    Dim r, i As Long
    r = Range("a1").CurrentRegion '数据装入数组
    For i = 2 To UBound(r)
    '标题行不要，从数组第二行开始遍历
        If r(i, 2) = "删除" Then Kill r(i, 1) 'Kill语句删除指定文件
    Next
    MsgBox "完成。"
End Sub
```

### 批量获取指定文件夹下文件名并创建超链接

```visual basic
Sub GetFiles()
    Dim strPath As String, strFileName As String, k As Long
    With Application.FileDialog(msoFileDialogFolderPicker)
        '用户选择文件夹路径
        If .Show Then strPath = .SelectedItems(1) Else Exit Sub
        '如果用户为选择文件夹则退出程序
    End With
    If Right(strPath, 1) <> "\" Then strPath = strPath & "\"
    Application.ScreenUpdating = False '取消屏幕刷新
    strFileName = Dir(strPath & "*.*")
    'dir+通配符获取首个文件名
    '如果一个文件也无，则返回空
    Columns(1).Clear: Cells(1, 1) = "目录": k = 1 '清除当前工作表A列数据
    Do While strFileName <> ""
        k = k + 1 '累加文件个数
        ActiveSheet.Hyperlinks.Add Cells(k, 1), strPath & strFileName
        '创建超链接
        strFileName = Dir
        '第2次调用Dir函数，未使用任何参数，则同目录下的下一个文件名
    Loop
    Application.ScreenUpdating = True
    MsgBox "一共读取了：" & k-1 & "个文件名。"
End Sub
```

### 批量给工作簿重命名

#### 获取文件夹下的工作簿

```visual basic
Sub GetFiles()
    Dim strPath As String, strFileName As String, k As Long
    With Application.FileDialog(msoFileDialogFolderPicker)
        If .Show Then strPath = .SelectedItems(1) Else: Exit Sub
        '获取用户选择的文件夹的路径，如果未选取，则退出程序
    End With
    If Right(strPath, 1) <> "\" Then strPath = strPath & "\"
    Application.ScreenUpdating = False
    Range("a:b").Clear: k = 1
    '清除A:B列的所有
    Cells(1, 1) = "旧文件名": Cells(1, 2) = "新文件名"
    strFileName = Dir(strPath & "*.xls*")
    Do While strFileName <> ""
        k = k + 1
        Cells(k, 1) = strPath & strFileName
        strFileName = Dir
    Loop
    Application.DisplayAlerts = True
End Sub
```

#### 重命名

```visual basic
Sub ChangeFileName()
    Dim r, i As Long
    r = Range("a1").CurrentRegion '数据装入数组
    For i = 2 To UBound(r)
    '标题行不要，从数组第二行开始遍历
        Name r(i, 1) As r(i, 2) 'Name语句重命名
    Next
    MsgBox "更名完成。"
End Sub
```

### 对office文件设置自杀程序

```visual basic
'此工具要生效，需要将Excel保存为【Excel启用宏的工作簿(*.xlsm)】格式
Private Sub Workbook_Open()
    Dim dat As Date
    dat = DateSerial(2020, 1, 1)
    If Date >= dat Then
        Application.DisplayAlerts = False
        MsgBox "你是在偷看我的文件吗？" & vbCr & "别以为我不知道，我就在你身后看着你！白衣服，长头发，没有腿的那个。"
        With ThisWorkbook
            .Saved = True
            .ChangeFileAccess xlReadOnly
            Kill .FullName
            .Close
        End With
    End If
End Sub
```

### 获取多层文件夹下文件名并创建超链接

```visual basic
Sub AutoAddLink()
    Dim strFldPath As String
    With Application.FileDialog(msoFileDialogFolderPicker)
    '用户选择指定文件夹
        .Title = "请选择指定文件夹。"
        If .Show Then strFldPath = .SelectedItems(1) Else Exit Sub
        '未选择文件夹则退出程序，否则将地址赋予变量strFldPath
    End With
    Application.ScreenUpdating = False
    '关闭屏幕刷新
    Range("a:b").ClearContents
    Range("a1:b1") = Array("文件夹", "文件名")
    Call SearchFileToHyperlinks(strFldPath)
    '调取自定义函数SearchFileToHyperlinks
    Range("a:b").EntireColumn.AutoFit
    '自动列宽
    Application.ScreenUpdating = True
    '重开屏幕刷新
End Sub
Function SearchFileToHyperlinks(ByVal strFldPath As String) As String
    Dim objFld As Object
    Dim objFile As Object
    Dim objSubFld As Object
    Dim strFilePath As String
    Dim lngLastRow As Long
    Dim intNum As Integer
    Set objFld = CreateObject("Scripting.FileSystemObject").GetFolder(strFldPath)
    '创建FileSystemObject对象引用
    For Each objFile In objFld.Files
    '遍历文件夹内的文件
        lngLastRow = Cells(Rows.Count, 1).End(xlUp).Row + 1
        strFilePath = objFile.Path
        intNum = InStrRev(strFilePath, "\")
        '使用instrrev函数获取最后文件夹名截至的位置
        Cells(lngLastRow, 1) = Left(strFilePath, intNum - 1)
        '文件夹地址
        Cells(lngLastRow, 2) = Mid(strFilePath, intNum + 1)
        '文件名
        ActiveSheet.Hyperlinks.Add Anchor:=Cells(lngLastRow, 2), _
                    Address:=strFilePath, ScreenTip:=strFilePath
        '添加超链接
    Next objFile
    For Each objSubFld In objFld.SubFolders
    '遍历文件夹内的子文件夹
        Call SearchFileToHyperlinks(objSubFld.Path)
    Next objSubFld
    Set objFld = Nothing
    Set objFile = Nothing
    Set objSubFld = Nothing
End Function
```

### 合并多工作簿数据成总表

```visual basic
'合并指定文件夹下多个工作簿数据
'1.代码运行后，首先会弹出一个选择文件夹的对话框，选中目标文件夹后，单击【确定】按钮即可。
'2.而后出现一个设置汇总工作表关键字的对话框。这里的关键字不区分大小写；如果需要汇总全部工作表的数据，可以不设置关键字，直接单击【确定】按钮即可。
'3.下一步设置工作表标题行的行数，默认为1行，可以根据实际情况设置为任意行，比如0行、2行等。
'4.数据合并完成后会弹出一个对话框，告知汇总了几张工作表的数据。
Sub CollectWorkBookDatas()
    Dim shtActive As Worksheet, rng As Range, shtData As Worksheet
    Dim nTitleRow As Long, k As Long, nLastRow As Long
    Dim i As Long, j As Long, nStartRow As Long
    Dim aData, aResult, nStarRng As Long
    Dim strPath As String, strFileName As String
    Dim strKey As String, nShtCount As Long
    With Application.FileDialog(msoFileDialogFolderPicker)
    '取得用户选择的文件夹路径
        If .Show Then strPath = .SelectedItems(1) Else Exit Sub
    End With
    If Right(strPath, 1) <> "\" Then strPath = strPath & "\"
    strKey = InputBox("请输入需要合并的工作表所包含的关键词：" & vbCrLf & "如未填写关键词，则默认汇总全部表格数据", "提醒")
    If StrPtr(strKey) = 0 Then Exit Sub '如果点击了取消或者关闭按钮，则退出程序
    nTitleRow = Val(InputBox("请输入标题的行数，默认标题行数为1", "提醒", 1))
    If nTitleRow < 0 Then MsgBox "标题行数不能为负数。", 64, "警告": Exit Sub
    Set shtActive = ActiveSheet
    With Application
        .ScreenUpdating = False
        .DisplayAlerts = False
        .AskToUpdateLinks = False
    End With
    ReDim aResult(1 To 80000, 1 To 1) '声明结果数组
    Cells.ClearContents '清空当前表格数据
    Cells.NumberFormat = "@" '设置单元格为文本格式
    strFileName = Dir(strPath & "*.xls*") '使用Dir函数遍历excel文件
    Do While strFileName <> ""
        If strFileName <> ThisWorkbook.Name Then '避免同名文件重复打开出错
            With GetObject(strPath & strFileName)
            '以只读'形式读取文件时，使用getobject会比workbooks.open稍快
                For Each shtData In .Worksheets '遍历表
                    If InStr(1, shtData.Name, strKey, vbTextCompare) Then
                    '如果表中包含关键字则进行汇总(不区分关键词字母大小写）
                        Set rng = shtData.UsedRange
                        If rng.Count > 1 Then '判断工作表是否存在数据……
                            nShtCount = nShtCount + 1 '汇总工作表的数量
                            nStartRow = IIf(nShtCount = 1, 1, nTitleRow + 1) '判断遍历数据源是否应该扣掉标题行
                            aData = rng.Value '数据区域读入数组arr
                            If UBound(aData, 2) + 2 > UBound(aResult, 2) Then '动态调整结果数组brr的最大列数
                                ReDim Preserve aResult(1 To UBound(aResult), 1 To UBound(aData, 2) + 2)
                            End If
                            For i = nStartRow To UBound(aData) '遍历行
                                k = k + 1
                                aResult(k, 1) = strFileName '数组第一列放工作簿名称
                                aResult(k, 2) = shtData.Name '数组第二列放工作表名称
                                For j = 1 To UBound(aData, 2) '遍历列
                                    aResult(k, j + 2) = aData(i, j)
                                Next
                                If k > UBound(aResult) - 1 Then
                                '如果数据行数到达结果数组的上限，则将数据导入汇总表，并清空结果数组
                                    With shtActive
                                        nLastRow = .Cells(Rows.Count, 1).End(xlUp).Row '获取放置来源数据的位置
                                        If nLastRow = 1 Then '判断是否扣除标题行
                                            nStarRng = IIf(nTitleRow = 0, 1, 0)
                                            .Range("a1").Offset(nStarRng).Resize(k, UBound(aResult, 2)) = aResult
                                            .Range("a1:b1") = Array("来源工作簿名称", "来源工作表名称")
                                            '前两列放来源工作簿和工作表名称
                                        Else
                                            .Range("a1").Offset(nLastRow).Resize(k, UBound(aResult, 2)) = aResult
                                            '放结果数组的数据
                                        End If
                                    End With
                                    k = 0
                                    ReDim aResult(1 To UBound(aResult), 1 To UBound(aResult, 2))
                                    '重新设置结果数组
                                End If
                            Next
                        End If
                    End If
                Next
                .Close False '关闭工作簿
            End With
        End If
        strFileName = Dir '下一个excel文件
    Loop
    If k > 0 Then
        shtActive.Select '激活汇总表
        nLastRow = Cells(Rows.Count, 1).End(xlUp).Row '放置数据的位置
        If nLastRow = 1 Then '如果汇总表数据为空，说明需要汇总的数据没有超过结果数组的上限
             nStarRng = IIf(nTitleRow = 0, 1, 0)
             Range("a1").Offset(nStarRng).Resize(k, UBound(aResult, 2)) = aResult
             Range("a1:b1") = Array("来源工作簿名称", "来源工作表名称")
         Else
             Range("a1").Offset(nLastRow).Resize(k, UBound(aResult, 2)) = aResult
         End If
    End If
    With Application
        .ScreenUpdating = True
        .DisplayAlerts = True
        .AskToUpdateLinks = True
    End With
    MsgBox "一共汇总完成。" & nShtCount & "个工作表", , "魏奇"
End Sub
```

### 将Word表格批量写入Excel

```visual basic
Sub GetWordTable()
    Dim WdApp As Object
    Dim objTable As Object
    Dim objDoc As Object
    Dim strPath As String
    Dim shtEach As Worksheet
    Dim shtSelect As Worksheet
    Dim i As Long
    Dim j As Long
    Dim x As Long
    Dim y As Long
    Dim k As Long
    Dim brr As Variant
    Set WdApp = CreateObject("Word.Application")
    With Application.FileDialog(msoFileDialogFilePicker)
        .Filters.Add "Word文件", "*.doc*", 1
        '只显示word文件
        .AllowMultiSelect = False
        '禁止多选文件
        If .Show Then strPath = .SelectedItems(1) Else Exit Sub
    End With
    Application.ScreenUpdating = False
    Application.DisplayAlerts = False
    Set shtSelect = ActiveSheet
    '当前表赋值变量shtSelect，方便代码运行完成后叶落归根回到开始的地方
    For Each shtEach In Worksheets
    '删除当前工作表以外的所有工作表
        If shtEach.Name <> shtSelect.Name Then shtEach.Delete
    Next
    shtSelect.Name = "魏奇"
    '这句代码不是无聊，作用在于……你猜……
    '……其实是避免下面的程序工作表名称重复
    Set objDoc = WdApp.documents.Open(strPath)
    '后台打开用户选定的word文档
    For Each objTable In objDoc.tables
    '遍历文档中的每个表格
        k = k + 1
        Worksheets.Add after:=Worksheets(Worksheets.Count)
        '新建工作表
        ActiveSheet.Name = k & "表"
        x = objTable.Rows.Count
        'table的行数
        y = objTable.Columns.Count
        'table的列数
        ReDim brr(1 To x, 1 To y)
        '以下遍历行列，数据写入数组brr
        For i = 1 To x
            For j = 1 To y
                brr(i, j) = "'" & Application.Clean(objTable.cell(i, j).Range.Text)
                'Clean函数清除制表符等
                '半角单引号将数据统一转换为文本格式，避免身份证等数值变形
            Next
        Next
        With [a1].Resize(x, y)
            .Value = brr
            '数据写入Excel工作表
            .Borders.LineStyle = 1
            '添加边框线
        End With
    Next
    shtSelect.Select
    objDoc.Close: WdApp.Quit
    Application.ScreenUpdating = True
    Application.DisplayAlerts = True
    Set objDoc = Nothing
    Set WdApp = Nothing
    MsgBox "共获取：" & k & "张表格的数据。"
End Sub
```

### 取消复杂的合并单元格

```visual basic
Sub UnMergeRange2() '取消合并单元格
Dim MaxRow As Integer '
Dim Rng As Range
Dim x%, y%, m%, n%, i%
Dim Rng2 As Range
    On Error Resume Next
    Set Rng = Application.InputBox("请选择需要取消合并单元格的区域：", _
                "区域选择", , , , , , 8)
    
    For x = 1 To Rng.Rows.Count
        For y = 1 To Rng.Columns.Count
            Set Rng2 = Rng.Cells(x, y)
            i = Rng2.MergeArea.Count
            If i > 1 Then
                m = Rng2.MergeArea.Rows.Count
                n = Rng2.MergeArea.Columns.Count
                Rng2.UnMerge '取消合并单元格
                Rng2.Resize(m, n).Value = Rng2.Value
            End If
        Next
    Next
End Sub
```

### 批量将图片插入到单元格批注中

```visual basic
Sub AddCommentPic()
    Dim arr, i&, k&, n&, b As Boolean
    Dim strPicName$, strPicPath$, strFdPath$
    Dim rngData As Range, rngEach As Range
    'On Error Resume Next
    '用户选择图片所在的文件夹
    With Application.FileDialog(msoFileDialogFolderPicker)
       If .Show Then strFdPath = .SelectedItems(1) Else: Exit Sub
    End With
    If Right(strFdPath, 1) <> "\" Then strFdPath = strFdPath & "\"
    Set rngData = Application.InputBox("请选择需要插入图片到批注中的单元格区域", Type:=8)
    '用户选择需要插入图片到批注中的单元格或区域
    If rngData.Count = 0 Then Exit Sub
    Set rngData = Intersect(rngData.Parent.UsedRange, rngData)
    'intersect语句避免用户选择整列单元格，造成无谓运算的情况
    If rngData Is Nothing Then MsgBox "选择单元格不能全为空。": Exit Sub
    arr = Array(".jpg", ".jpeg", ".bmp", ".png", ".gif")
    '用数组变量记录五种文件格式
    Application.ScreenUpdating = False
    For Each rngEach In rngData
    '遍历选择区域的每一个单元格
        If Not rngEach.Comment Is Nothing Then rngEach.Comment.Delete  '删除旧的批注
        strPicName = rngEach.Text '图片名称
        If Len(strPicName) Then '如果单元格存在值
            strPicPath = strFdPath & strPicName '图片路径
            b = False 'pd变量标记是否找到相关图片
            For i = 0 To UBound(arr)
            '由于不确定用户的图片格式，因此遍历图片格式
                If Len(Dir(strPicPath & arr(i))) Then
                '如果存在相关文件
                    rngEach.AddComment '增加批注
                    With rngEach.Comment
                        .Visible = True '批注可见
                        .Text Text:=""
                        .Shape.Select True '选中批注图形
                        Selection.ShapeRange.Fill.UserPicture strPicPath & arr(i)
                        '插入图片到批注中
                        .Shape.Height = 150 '图形的高度，可以根据需要自己调整
                        .Shape.Width = 150 '图形的宽度，可以根据需要自己调整
                        .Visible = False '取消显示
                    End With
                    b = True '标记找到结果
                    n = n + 1 '累加找到结果的个数
                    Exit For '找到结果后就可以退出文件格式循环
                End If
            Next
            If b = False Then k = k + 1  '如果没找到图片累加个数
        End If
    Next
    MsgBox "共处理成功" & n & "个图片，另有" & k & "个非空单元格未找到对应的图片。"
    Application.ScreenUpdating = True
End Sub
```

### 批量将图片插入到表格中

```visual basic
Sub InsertPic()
    Dim arr, i&, k&, n&, b As Boolean
    Dim strPicName$, strPicPath$, strFdPath$, shp As Shape
    Dim rngData As Range, rngEach As Range, rngWhere As Range, strWhere As String
    'On Error Resume Next
    '用户选择图片所在的文件夹
    With Application.FileDialog(msoFileDialogFolderPicker)
       If .Show Then strFdPath = .SelectedItems(1) Else: Exit Sub
    End With
    If Right(strFdPath, 1) <> "\" Then strFdPath = strFdPath & "\"
    Set rngData = Application.InputBox("请选择图片名称所在的单元格区域", Type:=8)
    '用户选择需要插入图片的名称所在单元格范围
    Set rngData = Intersect(rngData.Parent.UsedRange, rngData)
    'intersect语句避免用户选择整列单元格，造成无谓运算的情况
    If rngData Is Nothing Then MsgBox "选择的单元格范围不存在数据！": Exit Sub
    strWhere = InputBox("请输入图片偏移的位置，例如上1、下1、左1、右1", , "右1")
    '用户输入图片相对单元格的偏移位置。
    If Len(strWhere) = 0 Then Exit Sub
    x = Left(strWhere, 1)
    '偏移的方向
    If InStr("上下左右", x) = 0 Then MsgBox "你未输入偏移方位。": Exit Sub
    y = Val(Mid(strWhere, 2))
    '偏移的值
    Select Case x
        Case "上"
        Set rngWhere = rngData.Offset(-y, 0)
        Case "下"
        Set rngWhere = rngData.Offset(y, 0)
        Case "左"
        Set rngWhere = rngData.Offset(0, -y)
        Case "右"
        Set rngWhere = rngData.Offset(0, y)
    End Select
    Application.ScreenUpdating = False
    rngData.Parent.Parent.Activate '用户选定的激活工作簿
    rngData.Parent.Select
    For Each shp In ActiveSheet.Shapes
    '如果旧图片存放在目标图片存放范围则删除
        If Not Intersect(rngWhere, shp.TopLeftCell) Is Nothing Then shp.Delete
    Next
    x = rngWhere.Row - rngData.Row
    y = rngWhere.Column - rngData.Column
    '偏移的坐标
    arr = Array(".jpg", ".jpeg", ".bmp", ".png", ".gif")
    '用数组变量记录五种文件格式
    For Each rngEach In rngData
    '遍历选择区域的每一个单元格
        strPicName = rngEach.Text
        '图片名称
        If Len(strPicName) Then
        '如果单元格存在值
            strPicPath = strFdPath & strPicName
            '图片路径
            b = False
            '变量标记是否找到相关图片
            For i = 0 To UBound(arr)
            '由于不确定用户的图片格式，因此遍历图片格式
                If Len(Dir(strPicPath & arr(i))) Then
                '如果存在相关文件
                    Set shp = ActiveSheet.Shapes.AddPicture( _
                        strPicPath & arr(i), False, True, _
                        rngEach.Offset(x, y).Left + 5, _
                        rngEach.Offset(x, y).Top + 5, _
                        20, 20)
                    shp.Select
                    With Selection
                        .ShapeRange.LockAspectRatio = msoFalse
                        '撤销锁定图片纵横比
                        .Height = rngEach.Offset(x, y).Height - 10 '图片高度
                        .Width = rngEach.Offset(x, y).Width - 10 '图片宽度
                    End With
                    b = True '标记找到结果
                    n = n + 1 '累加找到结果的个数
                    Range("a1").Select: Exit For '找到结果后就可以退出文件格式循环
                End If
            Next
            If b = False Then k = k + 1 '如果没找到图片累加个数
        End If
    Next
    Application.ScreenUpdating = True
    MsgBox "共处理成功" & n & "个图片，另有" & k & "个非空单元格未找到对应的图片。"
End Sub
```

### 修改单元格内容会被记录到批注

```visual basic
'在所有过程之前用Dim语句定义的变量r1是模块级变量，应模块中所有的过程都可以使用它
Dim r1 '定义一个模块给变量，用户保存单元格的数据
'第一个事件过程，用于记录被更改前单元格中保存的数据
Private Sub Worksheet_SelectionChange(ByVal Target As Range)
If Target.Cells.Count <> 1 Then Exit Sub '选中多个单元格时退出程序
If Target.Formula = "" Then '根据选中单元格中保存的数据，确定给变量r1赋什么值
    r1 = "空"
Else
    r1 = Target.Text
End If
End Sub
'第二个事件过程，用于批注记录单元格修改前后的信息
Private Sub Worksheet_Change(ByVal Target As Range)
If Target.Cells.Count <> 1 Then Exit Sub
'定义变量保存单元格修改后的内容
Dim r2
'判断单元格是否被修改为空单元格
If Target.Formula = "" Then
    r2 = "空"
Else
    r2 = Target.Formula
End If
'如果单元格修改前后的内容一样则退出程序
If r1 = r2 Then Exit Sub
'定义一个批注变量
Dim r3
'定义一个变量保存批注内容
Dim r4
'将被修改单元格的批注赋给变量r3
Set r3 = Target.Comment
'如果单元格中没有批注则新建批注
If r3 Is Nothing Then Target.AddComment
'将批注的内容保存到变量r4中
r4 = Target.Comment.Text
'重新修改批注的内容=原批注内容+当前日期和时间+原内容+修改后的新内容
Target.Comment.Text Text:=r4 & Chr(10) & Format(Now(), "yyyy-mm-dd hh:mm") & "原内容:" & r1 & "修改为:" & r2
'根据批注内容自动调整批注大小
Target.Comment.Shape.TextFrame.AutoSize = True
End Sub
```

