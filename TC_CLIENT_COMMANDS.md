# TC Client Command Notes

這份文件記錄目前已驗證可用的 TC client 指令，範例目標是從 `ffxiv_dx11.7.05h.exe` 對到 `ffxiv_dx11.7.10.exe`。

所有指令都從專案根目錄執行：

```powershell
cd D:\project\opcodediff
```

## 1. 啟動 Python 環境

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy RemoteSigned
.\.venv\Scripts\Activate.ps1
```

確認 radare2 在 `PATH` 裡：

```powershell
radare2 -v
```

## 2. 如何取得 packet handler 與 switch 位址

TC client 的 binary 通常不會命中專案內建的 Global client signatures，所以要先用 radare2 找出兩個位址：

| 名稱 | 用途 |
| --- | --- |
| Packet handler address | `--packet-handler-addr`，也就是 zone packet dispatch handler 函式起點 |
| Switch offset/search address | `--packet-handler-switch-addr`，用來讓工具計算 opcode offset 並定位真正的 switch cases |

### 2.1 搜尋候選位址

先設定要分析的 exe：

```powershell
$exe = "ffxiv_dx11.7.05h.exe"
```

用下面這組 signatures 搜尋 handler prologue 和 switch 前置片段：

```powershell
radare2 -2 -q `
	-c "/x 48895c24..5556574154415541564157488dac24........b8........e8........482be0488b05........4833c4488985........450fb7" `
	-c "/x 4055535657415541564157488dac24........b8........e8........482be0488b05........4833c4488985........450fb778.." `
	-c "/x e8........4183c7..eb1b" `
	-c "q" $exe
```

三個 pattern 的意思：

| Pattern | 用途 |
| --- | --- |
| `48895c24..555657...450fb7` | 新版/TC 常見 packet handler prologue |
| `40555356574155...450fb778..` | 7.1 Global 風格 packet handler prologue，部分版本可命中 |
| `e8........4183c7..eb1b` | switch 前的 `call ...; add r15d, ...; jmp ...` 片段 |

如果輸出類似下面：

```text
0x14168d010 hit0_0 ...
0x14168dffd hit2_0 ...
```

代表：

| 結果 | 解讀 |
| --- | --- |
| `hit0_0` 或 `hit1_0` | packet handler 起點候選 |
| `hit2_0` | switch offset/search address 候選 |

### 2.2 反組譯確認 handler

用搜到的 handler 候選位址反組譯：

```powershell
radare2 -2 -q -c "s 0x14168d010" -c "pd 220" -c "q" $exe
```

高信心 packet handler 會有這些特徵：

```text
mov qword [rsp + ...], rbx
push rbp
push rsi
push rdi
push r12
push r13
push r14
push r15
lea rbp, [rsp - 0x1020]
...
movzx r15d, word [r8 + 2]
```

最重要的是 `movzx r15d, word [r8 + 2]`，這代表它從封包資料讀出 opcode。

如果 handler prologue 沒有命中，但 switch pattern 有命中，就從 switch 候選往前看：

```powershell
radare2 -2 -q -c "s 0x14168dffd" -c "pd -180" -c "pd 100" -c "q" $exe
```

往前找到第一個完整函式序言，通常就是 packet handler 起點。這次 7.10 就是先找到 switch，再往前確認 handler 起點。

### 2.3 反組譯確認 switch

用搜到的 switch 候選位址看前後：

```powershell
radare2 -2 -q -c "s 0x14168dffd" -c "pd -40" -c "pd 90" -c "q" $exe
```

高信心 switch 會像這樣：

```text
call 0x...
add r15d, 0xffffff9b
jmp 0x...
cmp r15d, 0x...
movsxd rax, r15d
mov ecx, dword [r12 + rax*4 + ...]
add rcx, r12
jmp rcx
```

`--packet-handler-switch-addr` 要填 `call 0x...` 那行的位址，也就是搜尋命中的 `hit2_0`。不要填 `jmp rcx` 那個真正 jump table switch 位址，因為工具需要從前面的 `call/add` 附近計算 opcode offset。

### 2.4 用暫存目錄驗證

找到兩個位址後，先用暫存目錄驗證：

```powershell
$check = Join-Path $env:TEMP ("opcodediff-check-" + [DateTime]::UtcNow.Ticks)
python generate_deep_traces.py --packet-handler-addr 0x14168d010 --packet-handler-switch-addr 0x14168dffd $exe $check
Get-ChildItem $check | Measure-Object | Select-Object Count
```

成功時會看到類似：

```text
Found opcode offset: 512
Found switch at 0x14168e031
Loaded 504 cases from packet handler
```

如果 `Loaded ... cases` 很少、找不到 switch、或 opcode offset 算出奇怪的值，通常是 switch address 填到真正 jump table 位置，或 handler 起點不是完整函式序言。

## 3. 產生 7.05h traces

7.05h TC client 已驗證位址：

| 項目 | 位址 |
| --- | --- |
| Packet handler | `0x14168d010` |
| Switch offset/search address | `0x14168dffd` |
| radare resolved switch | `0x14168e031` |
| Opcode offset | `512` |
| Cases | `504` |

如果要乾淨重產，先刪除舊輸出：

```powershell
Remove-Item -Recurse -Force 7.05h-traces -ErrorAction SilentlyContinue
```

產生 traces：

```powershell
python generate_deep_traces.py --packet-handler-addr 0x14168d010 --packet-handler-switch-addr 0x14168dffd ffxiv_dx11.7.05h.exe 7.05h-traces
```

預期輸出重點：

```text
Found opcode offset: 512
Found switch at 0x14168e031
Loaded 504 cases from packet handler
```

## 4. 產生 7.10 traces

7.10 TC client 已驗證位址：

| 項目 | 位址 |
| --- | --- |
| Packet handler | `0x14168bd10` |
| Switch offset/search address | `0x14168cd20` |
| radare resolved switch | `0x14168cd54` |
| Opcode offset | `512` |
| Cases | `504` |

如果要乾淨重產，先刪除舊輸出：

```powershell
Remove-Item -Recurse -Force 7.10-traces -ErrorAction SilentlyContinue
```

產生 traces：

```powershell
python generate_deep_traces.py --packet-handler-addr 0x14168bd10 --packet-handler-switch-addr 0x14168cd20 ffxiv_dx11.7.10.exe 7.10-traces
```

預期輸出重點：

```text
Found opcode offset: 512
Found switch at 0x14168cd54
Loaded 504 cases from packet handler
```

## 5. 產生 similarity matrix

```powershell
python generate_similarity_matrix.py 7.05h-traces 7.10-traces 7.10.similarity.json
```

## 6. 產生 diff

```powershell
python vtable_alignment.py ffxiv_dx11.7.05h.exe ffxiv_dx11.7.10.exe 7.10.similarity.json > 7.10.diff.json
```

## 7. 可選：輸出 opcode header 與 ACT format

如果要把 diff 轉成 opcode header：

```powershell
python generate_opcodes_file.py 7.05h 7.10 7.10.diff.json Ipcs.7.05h.h -o Ipcs.7.10.h
```

如果要再產生 ACT format：

```powershell
python generate_act_format.py Ipcs.7.10.h > act.7.10.txt
```

## 注意事項

- 7.05h 和 7.10 的 packet handler 位址不同，不要把 7.10 的位址拿去跑 7.05h。
- `--packet-handler-switch-addr` 要填 offset/search address，不是 radare 最後 resolved switch 位址。
- `generate_deep_traces.py` 會建立輸出資料夾並覆寫同名檔案；若要避免舊檔殘留，先用 `Remove-Item` 刪掉 traces 資料夾。