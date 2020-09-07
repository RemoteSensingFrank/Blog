---
title: excel与CSV格式的互相转换
date: 2018-01-31 22:14:52
tags: 效率
categories: 学习
---
话不多说，能够用代码解决的问题从来不用多说话，整个转换代码如下在excel中新建一个脚本，复制到VBA中修改文件所在的路径然后运行就可以了：
```VB
Sub SaveToXls()
    Dim fDir As String
    Dim wB As Workbook
    Dim wS As Worksheet
    Dim fPath As String
    Dim sPath As String
    Dim i As Integer
    i = 1
    Do While (i <= 18)
    fPath = "C:\Users\T51158\Desktop\point\3_132_Poletrans\" + CStr(i) + "\tower\"
    sPath = "C:\Users\T51158\Desktop\point\3_132_Poletrans\" + CStr(i) + "\xls\"
    fDir = Dir(fPath)
    Do While (fDir <> "")
        If Right(fDir, 4) = ".csv" Then
            On Error Resume Next
            Set wB = Workbooks.Open(fPath & fDir)
            'MsgBox (wB.Name)
            wB.SaveAs sPath & wB.Name & ".xlsx", FileFormat:=51
            wB.Close False
            Set wB = Nothing
        End If
        fDir = Dir
        On Error GoTo 0
    Loop
    i = i + 1
    Loop
End Sub

Sub SaveToCSVs()
    Dim fDir As String
    Dim wB As Workbook
    Dim wS As Worksheet
    Dim fPath As String
    Dim sPath As String
    fPath = "C:\Users\qiany\Desktop\文件\"
    sPath = "C:\Users\qiany\Desktop\csv保存位置\"
    fDir = Dir(fPath)
    Do While (fDir <> "")
        If Right(fDir, 4) = ".xls" Or Right(fDir, 5) = ".xlsx" Then
            On Error Resume Next
            Set wB = Workbooks.Open(fPath & fDir)
            'MsgBox (wB.Name)
            For Each wS In wB.Sheets
                wS.SaveAs sPath & wB.Name & ".csv", xlCSV
            Next wS
            wB.Close False
            Set wB = Nothing
        End If
        fDir = Dir
        On Error GoTo 0
    Loop
End Sub
```
