# **C-Life**

Персональное Android-приложение, которое собирает всё нужное для повседневной жизни в одном месте.

## Идея

Круто, что есть приложения на любой случай жизни. Но большинство из них выполняет одну маленькую задачу. Цель — **собрать** самые частые и полезные **функции в одном месте**.

**Принципы**

- **Скорость.** Записать событие быстрее, чем в заметках (или хотя бы так быстро, чтобы поймать вдохновение).
- **Отказоустойчивость.** Данные не должны теряться из-за сети, сбоев или закрытия приложения.

**Цель**

Я позиционирую этот проект как способ отработать свои навыки в проектировании ПО. Поэтому упор сделан на планирование, создание диаграм, схем, юзер-кейсов и т.д. и т.п.

## MVP

### Возможности

*У самурая нет цели, только путь.*

У меня нет видения того, как должно выглядеть готовое приложение. Знаю лишь, что оно должно быть модульным. В MVP планируется реализовать первый из них (задачи и счётчики).

* **Счётчики:** `+1` / `-1` по клику, история изменений, прогресс к цели.
* **Задачи:** разовые и повторяющиеся(по календарю или в зависимости от прошлого выполнения)

## Use-case диаграмма

![Use-case Diagram](//www.plantuml.com/plantuml/png/dLLHQnj747w_Np7ufJP5AMxVWcr2rT864YXnwgT0EUIDS_Ya4tVhfg4Kx8JMbAInOnoifetgs0zzxDcrgInRyb-O-qVzXcLrqcffZmHyEUtjD_FDpBVVPkjHgnscOPF2XaxIPVqaLlHb4iV9Omg2MvaEKxtHZ8DOFTIa4qgZjKUQLgDKDNIKj88ItmuRw-6QCX7aSzXQYrKGX1ANPlWF7l9byOB_mV-dd1TF-Ln-pr2OqTUPIeDW1AAPUkrEz53HxDsLh-PcwFiW8FpRo5GZp9J4AZPvM6pnbSJ1Uyz4B5uGvzn7Q5aO5C_vWh3hcdju3hd_e8NiEPzXxpPn3z-AxU8LSDi0z8MNOLF_PYLIZ-z4cJQ9Fw1QhKQynw_vYD_mBitV93v6HOUyYzNViFwME_oQ-1qMEtY-uIx_BYY7-35VWsxEfoXzHmXJyGjVeXDN-1k0U8wtNacwpxcbT3TieRr2IcAI4_WT3r7RcUt2gCpYcSNNKnLgTJ_CrXteM-pRaZOYHC_0hwKNO3F48dYXE-SFMhEoJ49966DL4cmMB-aZmiQd1CHpsOAl0uCgTkPizcMbxwcs2l-Vl3ELIVhrPAEbLUhW3pWlqJyPNSbymUPSr87mPsD4laZ2s4VZ4CWpAo3WZ4wdXsH3VRuQwTjQDTqPIn3ix_q7z3CLAwsyu1DazcM5U4-9fUC_Yhn7iaY-LVUJJt7uucJDBrVw678LgRxYBbPs8LFY0_pUnz-VsFMhHwc7gE54vZgQofYgmEZ24hWj0y4X5sNvkhfNgiAs10KQ6oYUGJvUiIRDjdHeEO8V-4Hx8A8LXPG4nL3wYFWpsdCXEf0pVOcsZQaPX-ubDe0-q3lZ5VL6gH-fnhhHQlr98rPEuYw0ICa32MeBsZ8wGp6UetD1inEJAy-4JjecqR9A8vNvfqUVJ4sFktXWSbjyX6V7wpR-6V9mslIkt4dQG-OPPWSTkp9rMQCPIFkiIz3iX_FmWzhyNDbEMBfQ2UEfCtzYJ9pzaIp-digK1jmsPt5nKOhVux-WWGvQko-z1-gKfJR8J5vpacWZintE2QuiIEuRz6l9P_LyR8w-404liVeZ6IxntmYUcui5bSeETBOlNz1YEQrzIIU9JP8oDYtSk37fuNIJ5XRKTrgrLfUMVDlTerKTK4vm4kEl3zumVYlPzezVGP28O9q2aBBqPGz3By0ryeeOwxGLjnjthGfmJRHAr-HCTlbQJ4RcB-sJMzlUOpmmwrZRz5qgvhk4Tbfe-9ItfPU_xKQ5tVAORdg9awkmQWxPMrLH-yPYKRDuvCxqxI5GL-AIos_-BaK_r6FUbO6k5xz_eLwBTLCxanYPf9-l7sISqSNy4Dp2oqOp_XS0)

## Схема базы данных

//www.plantuml.com/plantuml/png/pLDDYzH04BtdLnHx4GApU0xhifTHH12HENWBJNVjJ3CzwQ5J6Hrt1JyE7WJt9_WNH5rTLCQ_qFb7Tj8RAclcC8hYthhUUzNalNuvAaaugXOcCNXCG1QSdiu8b7OeITiYcMc58BMJ1fC42zAqXZqf2AVMhVT0bD0DS980f92cMgKfP30PFvh0mS63UuU7WHYDMhOG2xpamYojiIwVuvgX54hhABTEeGletVkJyPtnGxW5DrjIEXI4AXV4P4jKIxMDK6Xm03_hER5LGUYYaNZUmKTdETUA1RUZOAidqcIG0OMbT7hP1CpWr0gJat1Jl68XuajqsYeMnpbldu_LVpy_uUHCh_uurnnN0UdIPMINhdGRWv1ZekibzXDMQ4ZyuibAMJctpIqzvLivbz-tIQASj-wQm_ygZiCb2ievaHwecmHNml3dCrSLcWOBJQE6w6F4kHBhSa1K8GZ3Q85FA6-An90HPO1iWODi6Ri-xULdxLMNnrrw6-KxbFROkdvM_xYoDy2_gv_x2_-XFlDV_5V_7keNzSi0RVn7lm7_tP_xp_vJ_Ge2-Qr-Mx_svq7y9X0Nu7_uJG3FcY7fMfnbMi7fwVw-FUb-rYphLNohi4ddggfRZQBHY6-BeMTPDmPnN6TbCtUoo-2J8onKjJ0_0G00

подробнее в docs/arch.md

### Дальше

- [ ] Заметки дня (ежедневник, привязанный к дате)
- [ ] Хроника дня (?)
- [ ] Календарь (?)
- [ ] Интеграции с Notion(?) и Google Calendar(?)