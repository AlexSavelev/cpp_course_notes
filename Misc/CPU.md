# Cycles

![[../assets/CPU_cycles.jpg]]

- Каждая операция осуществляется за разное время
- При написании программы на C++ это стоит учитывать
- К слову, float division (r32) - это 22-29 cycles

| Cache    | Time (С++ Russia 2017)                          | Time (*book, page 16) |
| -------- | ----------------------------------------------- | --------------------- |
| Register |                                                 | $<= 1$ cycle(s)       |
| L1       | $5$ cycles (for complex address calcs (`p[n]`)) | ~$3$ (for L1d) cycles |
| L2       | $12$ cycles                                     | ~$14$ cycles          |
| L3       | $36-43$ cycles                                  |                       |
| Main mem |                                                 | ~$240$ cycles         |
- Вообще рекомендуется прочитать книгу "What Every Programmer Should Know About Memory", Ulrich Drepper, 2007 (`*`)
	- [Link](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf)

