# Work dependencies

Verified against native GitHub issue relationships on 2026-11-28. An arrow means the earlier task must finish before the later task. Epic grouping and tentative future order are separate from these blockers.

```mermaid
flowchart TD
  I1["#1 App and quantities"]
  I2["#2 Purchase accounting"]
  I3["#3 Offline storage"]
  I4["#4 Shopping screen"]
  I5["#5 Choose sign-in service"]
  I6["#6 Accounts and local privacy"]
  I7["#7 Households"]
  I8["#8 Exchange specification"]
  I9["#9 Server acceptance"]
  I10["#10 Phone sync"]
  I11["#11 Conflict resolution"]
  I12["#12 Catalog"]
  I13["#13 Units"]
  I14["#14 Lists and notes"]
  I15["#15 Delete and restore"]
  I16["#16 Notifications"]
  I17["#17 Account deletion"]
  I18["#18 End-to-end tests"]
  I19["#19 Beta release"]
  I18 --> I19
  I16 --> I18
  I17 --> I18
  I15 --> I17
  I7 --> I17
  I11 --> I16
  I6 --> I16
  I14 --> I15
  I13 --> I14
  I12 --> I13
  I11 --> I12
  I10 --> I11
  I9 --> I10
  I3 --> I10
  I8 --> I9
  I7 --> I9
  I6 --> I7
  I5 --> I6
  I3 --> I6
  I3 --> I4
  I2 --> I3
  I1 --> I2
  click I1 "https://github.com/Bazuzu77/shopping-list/issues/1"
  click I2 "https://github.com/Bazuzu77/shopping-list/issues/2"
  click I3 "https://github.com/Bazuzu77/shopping-list/issues/3"
  click I4 "https://github.com/Bazuzu77/shopping-list/issues/4"
  click I5 "https://github.com/Bazuzu77/shopping-list/issues/5"
  click I6 "https://github.com/Bazuzu77/shopping-list/issues/6"
  click I7 "https://github.com/Bazuzu77/shopping-list/issues/7"
  click I8 "https://github.com/Bazuzu77/shopping-list/issues/8"
  click I9 "https://github.com/Bazuzu77/shopping-list/issues/9"
  click I10 "https://github.com/Bazuzu77/shopping-list/issues/10"
  click I11 "https://github.com/Bazuzu77/shopping-list/issues/11"
  click I12 "https://github.com/Bazuzu77/shopping-list/issues/12"
  click I13 "https://github.com/Bazuzu77/shopping-list/issues/13"
  click I14 "https://github.com/Bazuzu77/shopping-list/issues/14"
  click I15 "https://github.com/Bazuzu77/shopping-list/issues/15"
  click I16 "https://github.com/Bazuzu77/shopping-list/issues/16"
  click I17 "https://github.com/Bazuzu77/shopping-list/issues/17"
  click I18 "https://github.com/Bazuzu77/shopping-list/issues/18"
  click I19 "https://github.com/Bazuzu77/shopping-list/issues/19"
```

Issue #8 has an uploaded approved design but still needs exact exchange formats and worked examples. It can progress without waiting for #2; their models must remain consistent. Future epics have no invented blockers or dates. Shared household timetable remains the farthest future item.

[Open the ordered project plan](https://github.com/users/Bazuzu77/projects/1/views/2) · [Backlog](README.md)

