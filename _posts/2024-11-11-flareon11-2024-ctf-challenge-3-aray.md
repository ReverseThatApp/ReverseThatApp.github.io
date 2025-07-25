---
layout: post
title: "Cracking the Flare-On 11 CTF 2024: Challenge 3 - aray"
image:
    path: https://lh3.googleusercontent.com/pw/AP1GczP0Zp7rfVdUh3LtetCVUK1JANiHG6tMP1IZJEq45yxABL23tTNMMWBJp5nK2wbZmb6PrmOhGjozDMpIWFEN-YFwNqQm4aNyjRSzFeXvnr84hq36rXextOfjYgWsek5tMpReUMkHUYLZ7AcjGQv0D7D-=w2284-h1354-s-no-gm?authuser=2
    alt: Challenge 3 - aray
tags: [flareon, flareon11, flareon-2024, ctf, yara]
categories: [CTF, Flareon 2024]
---
Given a `.yara` file with hundreds of conditions, we need to dive deep to recover each character and reconstruct the hidden Yara rules. This is indeed a pleasant challenge—or am I wrong?

## Challenge description
>3 - aray
>
>And now for something completely different. I'm pretty sure you know how to write Yara rules, but can you reverse them?
{: .prompt-info}

## Sanitize the Rules
Given an `array.yara` file with tons of rules, don’t feel dizzy at first glance. We need to sanitize it first and then gradually solve the puzzle. We can observe that all conditions in the file are concatenated with the **`and`** keyword, so we can use this to replace them with new line characters and sort conditions alphabetically to group related conditions together. Use the command `sort -n filename > replace_and_sorted_conditions.txt`. Here are the sanitized rules—`548 conditions` to be exact! ^_^


```
filesize == 85 
filesize ^ uint8(0) != 16 
filesize ^ uint8(0) != 41 
filesize ^ uint8(1) != 0 
filesize ^ uint8(1) != 232 
filesize ^ uint8(10) != 205 
filesize ^ uint8(10) != 44 
filesize ^ uint8(11) != 107 
filesize ^ uint8(11) != 33 
filesize ^ uint8(12) != 116 
filesize ^ uint8(12) != 226 
filesize ^ uint8(13) != 219 
filesize ^ uint8(13) != 42 
filesize ^ uint8(14) != 161 
filesize ^ uint8(14) != 99 
filesize ^ uint8(15) != 205 
filesize ^ uint8(15) != 27 
filesize ^ uint8(16) != 144 
filesize ^ uint8(16) != 7 
filesize ^ uint8(17) != 16 
filesize ^ uint8(17) != 208 
filesize ^ uint8(18) != 234 
filesize ^ uint8(18) != 33 
filesize ^ uint8(19) != 222 
filesize ^ uint8(19) != 31 
filesize ^ uint8(2) != 205 
filesize ^ uint8(2) != 54 
filesize ^ uint8(20) != 17 
filesize ^ uint8(20) != 83 
filesize ^ uint8(21) != 188 
filesize ^ uint8(21) != 27 
filesize ^ uint8(22) != 191 
filesize ^ uint8(22) != 31 
filesize ^ uint8(23) != 18 
filesize ^ uint8(23) != 242 
filesize ^ uint8(24) != 217 
filesize ^ uint8(24) != 94 
filesize ^ uint8(25) != 224 
filesize ^ uint8(25) != 47 
filesize ^ uint8(26) != 161 
filesize ^ uint8(26) != 44 
filesize ^ uint8(27) != 244 
filesize ^ uint8(27) != 43 
filesize ^ uint8(28) != 12 
filesize ^ uint8(28) != 238 
filesize ^ uint8(29) != 158 
filesize ^ uint8(29) != 37 
filesize ^ uint8(3) != 147 
filesize ^ uint8(3) != 43 
filesize ^ uint8(30) != 18 
filesize ^ uint8(30) != 249 
filesize ^ uint8(31) != 32 
filesize ^ uint8(31) != 5 
filesize ^ uint8(32) != 30 
filesize ^ uint8(32) != 77 
filesize ^ uint8(33) != 157 
filesize ^ uint8(33) != 27 
filesize ^ uint8(34) != 115 
filesize ^ uint8(34) != 39 
filesize ^ uint8(35) != 120 
filesize ^ uint8(35) != 18 
filesize ^ uint8(36) != 6 
filesize ^ uint8(36) != 95 
filesize ^ uint8(37) != 141 
filesize ^ uint8(37) != 37 
filesize ^ uint8(38) != 8 
filesize ^ uint8(38) != 84 
filesize ^ uint8(39) != 18 
filesize ^ uint8(39) != 49 
filesize ^ uint8(4) != 23 
filesize ^ uint8(4) != 253 
filesize ^ uint8(40) != 230 
filesize ^ uint8(40) != 49 
filesize ^ uint8(41) != 233 
filesize ^ uint8(41) != 74 
filesize ^ uint8(42) != 1 
filesize ^ uint8(42) != 91 
filesize ^ uint8(43) != 251 
filesize ^ uint8(43) != 33 
filesize ^ uint8(44) != 17 
filesize ^ uint8(44) != 96 
filesize ^ uint8(45) != 146 
filesize ^ uint8(45) != 19 
filesize ^ uint8(46) != 18 
filesize ^ uint8(46) != 186 
filesize ^ uint8(47) != 11 
filesize ^ uint8(47) != 119 
filesize ^ uint8(48) != 29 
filesize ^ uint8(48) != 99 
filesize ^ uint8(49) != 10 
filesize ^ uint8(49) != 156 
filesize ^ uint8(5) != 243 
filesize ^ uint8(5) != 43 
filesize ^ uint8(50) != 219 
filesize ^ uint8(50) != 86 
filesize ^ uint8(51) != 0 
filesize ^ uint8(51) != 204 
filesize ^ uint8(52) != 22 
filesize ^ uint8(52) != 238 
filesize ^ uint8(53) != 19 
filesize ^ uint8(53) != 243 
filesize ^ uint8(54) != 141 
filesize ^ uint8(54) != 39 
filesize ^ uint8(55) != 17 
filesize ^ uint8(55) != 244 
filesize ^ uint8(56) != 22 
filesize ^ uint8(56) != 246 
filesize ^ uint8(57) != 14 
filesize ^ uint8(57) != 186 
filesize ^ uint8(58) != 12 
filesize ^ uint8(58) != 77 
filesize ^ uint8(59) != 13 
filesize ^ uint8(59) != 194 
filesize ^ uint8(6) != 129 
filesize ^ uint8(6) != 39 
filesize ^ uint8(60) != 142 
filesize ^ uint8(60) != 43 
filesize ^ uint8(61) != 239 
filesize ^ uint8(61) != 94 
filesize ^ uint8(62) != 15 
filesize ^ uint8(62) != 246 
filesize ^ uint8(63) != 135 
filesize ^ uint8(63) != 34 
filesize ^ uint8(64) != 158 
filesize ^ uint8(64) != 50 
filesize ^ uint8(65) != 215 
filesize ^ uint8(65) != 28 
filesize ^ uint8(66) != 146 
filesize ^ uint8(66) != 51 
filesize ^ uint8(67) != 55 
filesize ^ uint8(67) != 63 
filesize ^ uint8(68) != 135 
filesize ^ uint8(68) != 8 
filesize ^ uint8(69) != 241 
filesize ^ uint8(69) != 30 
filesize ^ uint8(7) != 15 
filesize ^ uint8(7) != 221 
filesize ^ uint8(70) != 209 
filesize ^ uint8(70) != 41 
filesize ^ uint8(71) != 128 
filesize ^ uint8(71) != 3 
filesize ^ uint8(72) != 219 
filesize ^ uint8(72) != 37 
filesize ^ uint8(73) != 17 
filesize ^ uint8(73) != 61 
filesize ^ uint8(74) != 193 
filesize ^ uint8(74) != 45 
filesize ^ uint8(75) != 25 
filesize ^ uint8(75) != 35 
filesize ^ uint8(76) != 30 
filesize ^ uint8(76) != 88 
filesize ^ uint8(77) != 22 
filesize ^ uint8(77) != 223 
filesize ^ uint8(78) != 163 
filesize ^ uint8(78) != 6 
filesize ^ uint8(79) != 104 
filesize ^ uint8(79) != 186 
filesize ^ uint8(8) != 107 
filesize ^ uint8(8) != 2 
filesize ^ uint8(80) != 236 
filesize ^ uint8(80) != 56 
filesize ^ uint8(81) != 242 
filesize ^ uint8(81) != 7 
filesize ^ uint8(82) != 228 
filesize ^ uint8(82) != 32 
filesize ^ uint8(83) != 197 
filesize ^ uint8(83) != 31 
filesize ^ uint8(84) != 231 
filesize ^ uint8(84) != 3 
filesize ^ uint8(9) != 164 
filesize ^ uint8(9) != 5 
hash.crc32(34, 2) == 0x5888fc1b 
hash.crc32(63, 2) == 0x66715919 
hash.crc32(78, 2) == 0x7cab8d64 
hash.crc32(8, 2) == 0x61089c5c 
hash.md5(0, 2) == "89484b14b36a8d5329426a3d944d2983" 
hash.md5(0, filesize) == "b7dc94ca98aa58dabb5404541c812db2" 
hash.md5(32, 2) == "738a656e8e8ec272ca17cd51e12f558b" 
hash.md5(50, 2) == "657dae0913ee12be6fb2a6f687aae1c7" 
hash.md5(76, 2) == "f98ed07a4d5f50f7de1410d905f1477f" 
hash.sha256(14, 2) == "403d5f23d149670348b147a15eeb7010914701a7e99aad2e43f90cfa0325c76f" 
hash.sha256(56, 2) == "593f2d04aab251f60c9e4b8bbc1e05a34e920980ec08351a18459b2bc7dbf2f6" 
uint32(10) + 383041523 == 2448764514 
uint32(17) - 323157430 == 1412131772 
uint32(22) ^ 372102464 == 1879700858 
uint32(28) - 419186860 == 959764852 
uint32(3) ^ 298697263 == 2108416586 
uint32(37) + 367943707 == 1228527996 
uint32(41) + 404880684 == 1699114335 
uint32(46) - 412326611 == 1503714457 
uint32(52) ^ 425706662 == 1495724241 
uint32(59) ^ 512952669 == 1908304943 
uint32(66) ^ 310886682 == 849718389 
uint32(70) + 349203301 == 2034162376 
uint32(80) - 473886976 == 69677856 
uint8(0) % 25 < 25 
uint8(0) & 128 == 0 
uint8(0) < 129 
uint8(0) > 30 
uint8(1) % 17 < 17 
uint8(1) & 128 == 0 
uint8(1) < 158 
uint8(1) > 19 
uint8(10) % 10 < 10 
uint8(10) & 128 == 0 
uint8(10) < 146 
uint8(10) > 9 
uint8(11) % 27 < 27 
uint8(11) & 128 == 0 
uint8(11) < 154 
uint8(11) > 18 
uint8(12) % 23 < 23 
uint8(12) & 128 == 0 
uint8(12) < 147 
uint8(12) > 19 
uint8(13) % 27 < 27 
uint8(13) & 128 == 0 
uint8(13) < 147 
uint8(13) > 21 
uint8(14) % 19 < 19 
uint8(14) & 128 == 0 
uint8(14) < 153 
uint8(14) > 20 
uint8(15) % 16 < 16 
uint8(15) & 128 == 0 
uint8(15) < 156 
uint8(15) > 26 
uint8(16) % 31 < 31 
uint8(16) & 128 == 0 
uint8(16) < 134 
uint8(16) > 25 
uint8(16) ^ 7 == 115 
uint8(17) % 11 < 11 
uint8(17) & 128 == 0 
uint8(17) < 150 
uint8(17) > 31 
uint8(18) % 30 < 30 
uint8(18) & 128 == 0 
uint8(18) < 137 
uint8(18) > 13 
uint8(19) % 30 < 30 
uint8(19) & 128 == 0 
uint8(19) < 151 
uint8(19) > 4 
uint8(2) % 28 < 28 
uint8(2) & 128 == 0 
uint8(2) + 11 == 119 
uint8(2) < 147 
uint8(2) > 20 
uint8(20) % 28 < 28 
uint8(20) & 128 == 0 
uint8(20) < 135 
uint8(20) > 1 
uint8(21) % 11 < 11 
uint8(21) & 128 == 0 
uint8(21) - 21 == 94 
uint8(21) < 138 
uint8(21) > 7 
uint8(22) % 22 < 22 
uint8(22) & 128 == 0 
uint8(22) < 152 
uint8(22) > 20 
uint8(23) % 16 < 16 
uint8(23) & 128 == 0 
uint8(23) < 141 
uint8(23) > 2 
uint8(24) % 26 < 26 
uint8(24) & 128 == 0 
uint8(24) < 148 
uint8(24) > 22 
uint8(25) % 23 < 23 
uint8(25) & 128 == 0 
uint8(25) < 154 
uint8(25) > 27 
uint8(26) % 25 < 25 
uint8(26) & 128 == 0 
uint8(26) - 7 == 25 
uint8(26) < 132 
uint8(26) > 31 
uint8(27) % 26 < 26 
uint8(27) & 128 == 0 
uint8(27) < 147 
uint8(27) > 23 
uint8(27) ^ 21 == 40 
uint8(28) % 27 < 27 
uint8(28) & 128 == 0 
uint8(28) < 160 
uint8(28) > 27 
uint8(29) % 12 < 12 
uint8(29) & 128 == 0 
uint8(29) < 157 
uint8(29) > 22 
uint8(3) % 13 < 13 
uint8(3) & 128 == 0 
uint8(3) < 141 
uint8(3) > 21 
uint8(30) % 15 < 15 
uint8(30) & 128 == 0 
uint8(30) < 131 
uint8(30) > 6 
uint8(31) % 17 < 17 
uint8(31) & 128 == 0 
uint8(31) < 145 
uint8(31) > 7 
uint8(32) % 17 < 17 
uint8(32) & 128 == 0 
uint8(32) < 140 
uint8(32) > 28 
uint8(33) % 25 < 25 
uint8(33) & 128 == 0 
uint8(33) < 160 
uint8(33) > 18 
uint8(34) % 19 < 19 
uint8(34) & 128 == 0 
uint8(34) < 138 
uint8(34) > 18 
uint8(35) % 15 < 15 
uint8(35) & 128 == 0 
uint8(35) < 160 
uint8(35) > 1 
uint8(36) % 22 < 22 
uint8(36) & 128 == 0 
uint8(36) + 4 == 72 
uint8(36) < 146 
uint8(36) > 11 
uint8(37) % 19 < 19 
uint8(37) & 128 == 0 
uint8(37) < 139 
uint8(37) > 16 
uint8(38) % 24 < 24 
uint8(38) & 128 == 0 
uint8(38) < 135 
uint8(38) > 18 
uint8(39) % 11 < 11 
uint8(39) & 128 == 0 
uint8(39) < 134 
uint8(39) > 7 
uint8(4) % 17 < 17 
uint8(4) & 128 == 0 
uint8(4) < 139 
uint8(4) > 30 
uint8(40) % 19 < 19 
uint8(40) & 128 == 0 
uint8(40) < 131 
uint8(40) > 15 
uint8(41) % 27 < 27 
uint8(41) & 128 == 0 
uint8(41) < 140 
uint8(41) > 5 
uint8(42) % 17 < 17 
uint8(42) & 128 == 0 
uint8(42) < 157 
uint8(42) > 3 
uint8(43) % 26 < 26 
uint8(43) & 128 == 0 
uint8(43) < 160 
uint8(43) > 24 
uint8(44) % 27 < 27 
uint8(44) & 128 == 0 
uint8(44) < 147 
uint8(44) > 5 
uint8(45) % 17 < 17 
uint8(45) & 128 == 0 
uint8(45) < 136 
uint8(45) > 17 
uint8(45) ^ 9 == 104 
uint8(46) % 28 < 28 
uint8(46) & 128 == 0 
uint8(46) < 154 
uint8(46) > 22 
uint8(47) % 18 < 18 
uint8(47) & 128 == 0 
uint8(47) < 142 
uint8(47) > 13 
uint8(48) % 12 < 12 
uint8(48) & 128 == 0 
uint8(48) < 136 
uint8(48) > 15 
uint8(49) % 13 < 13 
uint8(49) & 128 == 0 
uint8(49) < 129 
uint8(49) > 27 
uint8(5) % 27 < 27 
uint8(5) & 128 == 0 
uint8(5) < 158 
uint8(5) > 14 
uint8(50) % 11 < 11 
uint8(50) & 128 == 0 
uint8(50) < 138 
uint8(50) > 19 
uint8(51) % 15 < 15 
uint8(51) & 128 == 0 
uint8(51) < 139 
uint8(51) > 7 
uint8(52) % 23 < 23 
uint8(52) & 128 == 0 
uint8(52) < 136 
uint8(52) > 25 
uint8(53) % 23 < 23 
uint8(53) & 128 == 0 
uint8(53) < 144 
uint8(53) > 24 
uint8(54) % 25 < 25 
uint8(54) & 128 == 0 
uint8(54) < 152 
uint8(54) > 15 
uint8(55) % 11 < 11 
uint8(55) & 128 == 0 
uint8(55) < 153 
uint8(55) > 5 
uint8(56) % 26 < 26 
uint8(56) & 128 == 0 
uint8(56) < 155 
uint8(56) > 8 
uint8(57) % 27 < 27 
uint8(57) & 128 == 0 
uint8(57) < 138 
uint8(57) > 11 
uint8(58) % 14 < 14 
uint8(58) & 128 == 0 
uint8(58) + 25 == 122 
uint8(58) < 146 
uint8(58) > 30 
uint8(59) % 23 < 23 
uint8(59) & 128 == 0 
uint8(59) < 141 
uint8(59) > 4 
uint8(6) % 12 < 12 
uint8(6) & 128 == 0 
uint8(6) < 155 
uint8(6) > 6 
uint8(60) % 23 < 23 
uint8(60) & 128 == 0 
uint8(60) < 130 
uint8(60) > 14 
uint8(61) % 26 < 26 
uint8(61) & 128 == 0 
uint8(61) < 160 
uint8(61) > 12 
uint8(62) % 13 < 13 
uint8(62) & 128 == 0 
uint8(62) < 146 
uint8(62) > 1 
uint8(63) % 30 < 30 
uint8(63) & 128 == 0 
uint8(63) < 129 
uint8(63) > 31 
uint8(64) % 24 < 24 
uint8(64) & 128 == 0 
uint8(64) < 154 
uint8(64) > 27 
uint8(65) % 22 < 22 
uint8(65) & 128 == 0 
uint8(65) - 29 == 70 
uint8(65) < 149 
uint8(65) > 1 
uint8(66) % 16 < 16 
uint8(66) & 128 == 0 
uint8(66) < 133 
uint8(66) > 30 
uint8(67) % 16 < 16 
uint8(67) & 128 == 0 
uint8(67) < 144 
uint8(67) > 27 
uint8(68) % 19 < 19 
uint8(68) & 128 == 0 
uint8(68) < 138 
uint8(68) > 10 
uint8(69) % 30 < 30 
uint8(69) & 128 == 0 
uint8(69) < 148 
uint8(69) > 25 
uint8(7) % 12 < 12 
uint8(7) & 128 == 0 
uint8(7) - 15 == 82 
uint8(7) < 131 
uint8(7) > 18 
uint8(70) % 21 < 21 
uint8(70) & 128 == 0 
uint8(70) < 139 
uint8(70) > 6 
uint8(71) % 28 < 28 
uint8(71) & 128 == 0 
uint8(71) < 130 
uint8(71) > 19 
uint8(72) % 14 < 14 
uint8(72) & 128 == 0 
uint8(72) < 134 
uint8(72) > 10 
uint8(73) % 23 < 23 
uint8(73) & 128 == 0 
uint8(73) < 136 
uint8(73) > 26 
uint8(74) % 10 < 10 
uint8(74) & 128 == 0 
uint8(74) + 11 == 116 
uint8(74) < 152 
uint8(74) > 1 
uint8(75) % 24 < 24 
uint8(75) & 128 == 0 
uint8(75) - 30 == 86 
uint8(75) < 142 
uint8(75) > 30 
uint8(76) % 24 < 24 
uint8(76) & 128 == 0 
uint8(76) < 156 
uint8(76) > 2 
uint8(77) % 24 < 24 
uint8(77) & 128 == 0 
uint8(77) < 154 
uint8(77) > 5 
uint8(78) % 13 < 13 
uint8(78) & 128 == 0 
uint8(78) < 141 
uint8(78) > 24 
uint8(79) % 24 < 24 
uint8(79) & 128 == 0 
uint8(79) < 146 
uint8(79) > 31 
uint8(8) % 21 < 21 
uint8(8) & 128 == 0 
uint8(8) < 133 
uint8(8) > 3
uint8(80) % 31 < 31 
uint8(80) & 128 == 0 
uint8(80) < 143 
uint8(80) > 2 
uint8(81) % 14 < 14 
uint8(81) & 128 == 0 
uint8(81) < 131 
uint8(81) > 11 
uint8(82) % 28 < 28 
uint8(82) & 128 == 0 
uint8(82) < 152 
uint8(82) > 3 
uint8(83) % 21 < 21 
uint8(83) & 128 == 0 
uint8(83) < 134 
uint8(83) > 16 
uint8(84) % 18 < 18 
uint8(84) & 128 == 0 
uint8(84) + 3 == 128 
uint8(84) < 129 
uint8(84) > 26 
uint8(9) % 22 < 22 
uint8(9) & 128 == 0 
uint8(9) < 151 
uint8(9) > 23
```
{: file='replace_and_sorted_conditions.txt'}

## Understand the Condition Format

First of all, the file size must be 85 bytes (`filesize == 85`). Each byte index (zero-based) is enclosed in parentheses. For example, `uint8(10) > 9` is a condition that checks if the byte at index `10` is greater than `9`, where `uint32(10)` refers to 4 bytes starting from byte index `10`. You might wonder whether it’s little-endian or big-endian? Good question! By default, `intXX` functions are **little-endian**.

## File Content Table

Let’s create a template to indicate the byte indices of the file and the values they should hold. We will fill in the values one by one based on the conditions.


| Index | 0  | 1  | 2  | 3  | 4  | 5  | 6  | 7  | 8  | 9  | 10 | 11 | 12 | 13 | 14 | 15 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |

| Index | 16 | 17 | 18 | 19 | 20 | 21 | 22 | 23 | 24 | 25 | 26 | 27 | 28 | 29 | 30 | 31 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |

| Index | 32 | 33 | 34 | 35 | 36 | 37 | 38 | 39 | 40 | 41 | 42 | 43 | 44 | 45 | 46 | 47 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |

| Index | 48 | 49 | 50 | 51 | 52 | 53 | 54 | 55 | 56 | 57 | 58 | 59 | 60 | 61 | 62 | 63 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |

| Index | 64 | 65 | 66 | 67 | 68 | 69 | 70 | 71 | 72 | 73 | 74 | 75 | 76 | 77 | 78 | 79 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |

| Index | 80 | 81 | 82 | 83 | 84 |    |    |    |    |    |    |    |    |    |    |    |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |    |


## Which Byte Indices Should Be a Good Start?

Let’s scan through the conditions from top to bottom and pick out conditions on byte indices that can be deduced to concrete values. We can identify a few:

```
uint32(10) + 383041523 == 2448764514 
uint32(17) - 323157430 == 1412131772 
uint32(22) ^ 372102464 == 1879700858 
uint32(28) - 419186860 == 959764852 
uint32(3) ^ 298697263 == 2108416586 
uint32(37) + 367943707 == 1228527996 
uint32(41) + 404880684 == 1699114335 
uint32(46) - 412326611 == 1503714457 
uint32(52) ^ 425706662 == 1495724241 
uint32(59) ^ 512952669 == 1908304943 
uint32(66) ^ 310886682 == 849718389 
uint32(70) + 349203301 == 2034162376 
uint32(80) - 473886976 == 69677856 
```

### uint32(index)

Let’s take this condition as an example: `uint32(10) + 383041523 == 2448764514`

This is an easy start, as we can deduce that `uint32(10) = 2448764514 - 383041523 = 2065722991 = 0x7B206E6F`.

Notice that the result represents 4 bytes starting from byte index `10` (the 11th byte). Remember, this is in little-endian format, so we get our first 4 bytes' value as follows:

```
byte_arr[10] = 0x6F = 'o'
byte_arr[11] = 0x6E = 'n'
byte_arr[12] = 0x20 = ' '
byte_arr[13] = 0x7B = '{'
```

Let’s take another condition: `uint32(3) ^ 298697263 == 2108416586`. From this, we get `uint32(3) = 2108416586 ^ 298697263 = 1818632293 = 0x6C662065`.

```
byte_arr[3] = 0x65 = 'e'
byte_arr[4] = 0x20 = ' '
byte_arr[5] = 0x66 = 'f'
byte_arr[6] = 0x6C = 'l'
```

Applying the same computation to other `uint32(x)` functions, we identified a few byte values that we can now fill into the table:

```
uint32(10) + 383041523 == 2448764514    => uint32(10) = 0x7B206E6F = "{ no"
uint32(17) - 323157430 == 1412131772    => uint32(17) = 0x676E6972 = "gnir"
uint32(22) ^ 372102464 == 1879700858    => uint32(22) = 0x6624203A = "f$ :"
uint32(28) - 419186860 == 959764852     => uint32(28) = 0x52312220 = "R1" "
uint32(3) ^ 298697263 == 2108416586     => uint32(3)  = 0x6C662065 = "lf e"
uint32(37) + 367943707 == 1228527996    => uint32(37) = 0x334B7961 = "3Kya"
uint32(41) + 404880684 == 1699114335    => uint32(41) = 0x4D247033 = "M$p3"
uint32(46) - 412326611 == 1503714457    => uint32(46) = 0x7234776C = "r4wl"
uint32(52) ^ 425706662 == 1495724241    => uint32(52) = 0x6F2D6572 = "@y4w"
uint32(59) ^ 512952669 == 1908304943    => uint32(59) = 0x6F2D6572 = "o-er"
uint32(66) ^ 310886682 == 849718389     => uint32(66) = 0x20226D6F = " "mo"
uint32(70) + 349203301 == 2034162376    => uint32(70) = 0x646E6F63 = "dnoc"
uint32(80) - 473886976 == 69677856      => uint32(80) = 0x20662420 = " f$ "
```


| Index | 0  | 1  | 2  | 3  | 4  | 5  | 6  | 7  | 8  | 9  | 10 | 11 | 12 | 13 | 14 | 15 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |    |    |    |'e' |' ' |'f' |'l' |    |    |    |'o' |'n' |' ' |'{' |    |    |

| Index | 16 | 17 | 18 | 19 | 20 | 21 | 22 | 23 | 24 | 25 | 26 | 27 | 28 | 29 | 30 | 31 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |    |'r' |'i' |'n' |'g' |    |':' |' ' |'$' |'f' |    |    |' ' |'"' |'1' |'R' |

| Index | 32 | 33 | 34 | 35 | 36 | 37 | 38 | 39 | 40 | 41 | 42 | 43 | 44 | 45 | 46 | 47 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |    |    |    |    |    |'a' |'y' |'K' |'3' |'3' |'p' |'$' |'M' |    |'l' |'w' |

| Index | 48 | 49 | 50 | 51 | 52 | 53 | 54 | 55 | 56 | 57 | 58 | 59 | 60 | 61 | 62 | 63 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |'4' |'r' |    |    |'w' |'4' |'y' |'@' |    |    |    |'r' |'e' |'-' |'o' |    |

| Index | 64 | 65 | 66 | 67 | 68 | 69 | 70 | 71 | 72 | 73 | 74 | 75 | 76 | 77 | 78 | 79 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |    |    |'o' |'m' |'"' |' ' |'c' |'o' |'n' |'d' |    |    |    |    |    |    |

| Index | 80 | 81 | 82 | 83 | 84 |    |    |    |    |    |    |    |    |    |    |    |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |' ' |'$' |'f' |' ' |    |    |    |    |    |    |    |    |    |    |    |    |

## Predict the Missing Bytes

As we can see, part of the flag is starting to appear. We can observe an `"@"` followed by 3 bytes, then `"re-o"`, another 3 bytes, and finally `"om"`. It’s reasonable to guess that this pattern corresponds to `"@flare-on.com"`. Let’s update the table accordingly.

| Index | 0  | 1  | 2  | 3  | 4  | 5  | 6  | 7  | 8  | 9  | 10 | 11 | 12 | 13 | 14 | 15 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |    |    |    |'e' |' ' |'f' |'l' |    |    |    |'o' |'n' |' ' |'{' |    |    |

| Index | 16 | 17 | 18 | 19 | 20 | 21 | 22 | 23 | 24 | 25 | 26 | 27 | 28 | 29 | 30 | 31 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |    |'r' |'i' |'n' |'g' |    |':' |' ' |'$' |'f' |    |    |' ' |'"' |'1' |'R' |

| Index | 32 | 33 | 34 | 35 | 36 | 37 | 38 | 39 | 40 | 41 | 42 | 43 | 44 | 45 | 46 | 47 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |    |    |    |    |    |'a' |'y' |'K' |'3' |'3' |'p' |'$' |'M' |    |'l' |'w' |

| Index | 48 | 49 | 50 | 51 | 52 | 53 | 54 | 55 | 56 | 57 | 58 | 59 | 60 | 61 | 62 | 63 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |'4' |'r' |    |    |'w' |'4' |'y' |'@' |'f' |'l' |'a' |'r' |'e' |'-' |'o' |'n' |

| Index | 64 | 65 | 66 | 67 | 68 | 69 | 70 | 71 | 72 | 73 | 74 | 75 | 76 | 77 | 78 | 79 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |'.' |'c' |'o' |'m' |'"' |' ' |'c' |'o' |'n' |'d' |    |    |    |    |    |    |

| Index | 80 | 81 | 82 | 83 | 84 |    |    |    |    |    |    |    |    |    |    |    |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |' ' |'$' |'f' |' ' |    |    |    |    |    |    |    |    |    |    |    |    |

With this updated table, we can better identify where the flag starts and ends. It begins at byte index `30` and ends at byte index `67`. Our flag appears to be enclosed in double quotes, meaning we can focus solely on the missing bytes within the range `[30:67]`. The rest of the bytes can likely be ignored, as they aren’t part of the flag—unless, of course, you’re curious enough to fully recover the entire byte content! :D

## Fill in the Holes

Here are the remaining byte indices within the flag range that we haven’t discovered yet: `32, 33, 34, 35, 36, 45, 50, 51`. These 8 missing bytes are significantly fewer compared to the hundreds of conditions we haven’t touched, so this looks promising.

If you’re a fan of Pokémon, we might be on the same wavelength: think of this as a Pokédex, and the missing indices as Pokémon we need to catch—Gotta Catch 'Em All! ^_^

It’s tempting to start with these indices in order, but there’s a better strategy: let’s pick the indices adjacent to known bytes, as we can make educated guesses based on surrounding characters. Let’s start with byte index `50`.

### Byte Index `50` - MD5 Checksum

Using the `grep` command with `"(50"` reveals 7 conditions applied to this byte index.

```bash
$ grep "(50" replaced_and_sorted_conditions.txt
filesize ^ uint8(50) != 219
filesize ^ uint8(50) != 86
hash.md5(50, 2) == "657dae0913ee12be6fb2a6f687aae1c7"
uint8(50) % 11 < 11
uint8(50) & 128 == 0
uint8(50) < 138
uint8(50) > 19
```

Let tackle them one by one:
```
filesize ^ uint8(50) != 219 
    => uint8(50) != filesize ^ 219 
    => uint8(50) != 85 ^ 219 
    => uint8(50) != 142
    => This condition is always true, as the flag consists of printable characters, so the range is between `32 to 126` (or `0x20 to 0x7E`).

```

```
filesize ^ uint8(50) != 86
    => uint8(50) != 85 ^ 86
    => uint8(50) != 3
    => Similarly, this condition is always true.
```

```
uint8(50) % 11 < 11
    => This condition is always true for any bytes in the range `0x20` to `0x7E`, as the remainder when dividing by 11 will always fall between 0 and 10.
```

```
uint8(50) & 128 == 0
    => This means the highest bit of this byte (bit 7) must be 0, indicating that the byte is less than 128. This condition is always true.
```

```
uint8(50) < 138
    => This condition is always true.
```

```
uint8(50) > 19 
    => This condition is always true.
```

Basically, all the above conditions are useless in narrowing down the possible range for byte index `50`. However, we overlooked one remaining condition: `hash.md5(50, 2) == "657dae0913ee12be6fb2a6f687aae1c7"`. This `hash.md5(offset, length)` function calculates the MD5 hash of a substring of the flag. In this case, it takes the bytes starting at offset 50 and reads 2 bytes, meaning bytes `50` and `51`. 

With this condition, we can consider brute-forcing to find a pair of bytes that produce the same hash. We can quickly write a Python script to brute-force this pair:
```bash
$ python3
Python 3.13.0 (main, Oct  7 2024, 05:02:14) [Clang 15.0.0 (clang-1500.3.9.4)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> import hashlib
... def brute_force_md5(target_md5):
...     for b1 in range(20, 127):
...         for b2 in range(20, 127):
...             data = bytearray([b1, b2])
...             digest = hashlib.md5(data).hexdigest()
...             if digest == target_md5:
...                 print(f"MD5 matched bytes: {b1}('{chr(b1)}'), {b2}('{chr(b2)}')")
...                 return
...     print("No valid bytes found!!!")
...
... brute_force_md5("657dae0913ee12be6fb2a6f687aae1c7")
...
MD5 matched bytes: 51('3'), 65('A')
```

We use two loops that iterate over the ASCII range of printable characters, hash each pair, and compare it with the target digest `"657dae0913ee12be6fb2a6f687aae1c7"`. With only two bytes to brute-force within a small range, it completes in about a second. We found that the values for bytes `50-51` are `"3A"`. Let’s update the table accordingly.

| Index | 0  | 1  | 2  | 3  | 4  | 5  | 6  | 7  | 8  | 9  | 10 | 11 | 12 | 13 | 14 | 15 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |    |    |    |'e' |' ' |'f' |'l' |    |    |    |'o' |'n' |' ' |'{' |    |    |

| Index | 16 | 17 | 18 | 19 | 20 | 21 | 22 | 23 | 24 | 25 | 26 | 27 | 28 | 29 | 30 | 31 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |    |'r' |'i' |'n' |'g' |    |':' |' ' |'$' |'f' |    |    |' ' |'"' |'1' |'R' |

| Index | 32 | 33 | 34 | 35 | 36 | 37 | 38 | 39 | 40 | 41 | 42 | 43 | 44 | 45 | 46 | 47 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |    |    |    |    |    |'a' |'y' |'K' |'3' |'3' |'p' |'$' |'M' |    |'l' |'w' |

| Index | 48 | 49 | 50 | 51 | 52 | 53 | 54 | 55 | 56 | 57 | 58 | 59 | 60 | 61 | 62 | 63 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |'4' |'r' |'3' |'A' |'w' |'4' |'y' |'@' |'f' |'l' |'a' |'r' |'e' |'-' |'o' |'n' |

| Index | 64 | 65 | 66 | 67 | 68 | 69 | 70 | 71 | 72 | 73 | 74 | 75 | 76 | 77 | 78 | 79 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |'.' |'c' |'o' |'m' |'"' |' ' |'c' |'o' |'n' |'d' |    |    |    |    |    |    |

| Index | 80 | 81 | 82 | 83 | 84 |    |    |    |    |    |    |    |    |    |    |    |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |' ' |'$' |'f' |' ' |    |    |    |    |    |    |    |    |    |    |    |    |


### Byte Index `32` - MD5 Checksum

Repeating the same process as in the previous step, we find that an MD5 hash is also applied to bytes `32-33`: `hash.md5(32, 2) == "738a656e8e8ec272ca17cd51e12f558b"`.

Running the Python brute-force helper with this new digest `brute_force_md5("738a656e8e8ec272ca17cd51e12f558b")`, we find that the byte values are `"ul"`.

| Index | 0  | 1  | 2  | 3  | 4  | 5  | 6  | 7  | 8  | 9  | 10 | 11 | 12 | 13 | 14 | 15 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |    |    |    |'e' |' ' |'f' |'l' |    |    |    |'o' |'n' |' ' |'{' |    |    |

| Index | 16 | 17 | 18 | 19 | 20 | 21 | 22 | 23 | 24 | 25 | 26 | 27 | 28 | 29 | 30 | 31 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |    |'r' |'i' |'n' |'g' |    |':' |' ' |'$' |'f' |    |    |' ' |'"' |'1' |'R' |

| Index | 32 | 33 | 34 | 35 | 36 | 37 | 38 | 39 | 40 | 41 | 42 | 43 | 44 | 45 | 46 | 47 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |'u' |'l' |    |    |    |'a' |'y' |'K' |'3' |'3' |'p' |'$' |'M' |    |'l' |'w' |

| Index | 48 | 49 | 50 | 51 | 52 | 53 | 54 | 55 | 56 | 57 | 58 | 59 | 60 | 61 | 62 | 63 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |'4' |'r' |'3' |'A' |'w' |'4' |'y' |'@' |'f' |'l' |'a' |'r' |'e' |'-' |'o' |'n' |

| Index | 64 | 65 | 66 | 67 | 68 | 69 | 70 | 71 | 72 | 73 | 74 | 75 | 76 | 77 | 78 | 79 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |'.' |'c' |'o' |'m' |'"' |' ' |'c' |'o' |'n' |'d' |    |    |    |    |    |    |

| Index | 80 | 81 | 82 | 83 | 84 |    |    |    |    |    |    |    |    |    |    |    |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |' ' |'$' |'f' |' ' |    |    |    |    |    |    |    |    |    |    |    |    |

The flag is almost revealed: `1Rul???ayK33p$M?lw4r3Aw4y@flare-on.com`. We have only 4 characters remaining (actually just 2, as we can guess the flag might be `1 rule something keeps malware away`).

### Byte Index `34` - CRC32 Checksum

Repeating the same steps as before, we find that this byte is involved in a CRC32 checksum function: `hash.crc32(34, 2) == 0x5888fc1b`, which involves byte indices `34` and `35`.
```bashscript
$ python3
Python 3.13.0 (main, Oct  7 2024, 05:02:14) [Clang 15.0.0 (clang-1500.3.9.4)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> import zlib
... def brute_force_crc32(target_crc32):
...     for b1 in range(20, 127):
...         for b2 in range(20, 127):
...             data = bytearray([b1, b2])
...             checksum = zlib.crc32(data)
...             if checksum == target_crc32:
...                 print(f"CRC32 matched bytes: {b1}('{chr(b1)}'), {b2}('{chr(b2)}')")
...                 return
...     print("No valid bytes found!!!")
...
... brute_force_crc32(0x5888fc1b)
...
CRC32 matched bytes: 101('e'), 65('A')
```

We found the byte values are `"eA"`. Let’s update the table accordingly.

| Index | 0  | 1  | 2  | 3  | 4  | 5  | 6  | 7  | 8  | 9  | 10 | 11 | 12 | 13 | 14 | 15 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |    |    |    |'e' |' ' |'f' |'l' |    |    |    |'o' |'n' |' ' |'{' |    |    |

| Index | 16 | 17 | 18 | 19 | 20 | 21 | 22 | 23 | 24 | 25 | 26 | 27 | 28 | 29 | 30 | 31 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |    |'r' |'i' |'n' |'g' |    |':' |' ' |'$' |'f' |    |    |' ' |'"' |'1' |'R' |

| Index | 32 | 33 | 34 | 35 | 36 | 37 | 38 | 39 | 40 | 41 | 42 | 43 | 44 | 45 | 46 | 47 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |'u' |'l' |'e' |'A' |    |'a' |'y' |'K' |'3' |'3' |'p' |'$' |'M' |    |'l' |'w' |

| Index | 48 | 49 | 50 | 51 | 52 | 53 | 54 | 55 | 56 | 57 | 58 | 59 | 60 | 61 | 62 | 63 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |'4' |'r' |'3' |'A' |'w' |'4' |'y' |'@' |'f' |'l' |'a' |'r' |'e' |'-' |'o' |'n' |

| Index | 64 | 65 | 66 | 67 | 68 | 69 | 70 | 71 | 72 | 73 | 74 | 75 | 76 | 77 | 78 | 79 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |'.' |'c' |'o' |'m' |'"' |' ' |'c' |'o' |'n' |'d' |    |    |    |    |    |    |

| Index | 80 | 81 | 82 | 83 | 84 |    |    |    |    |    |    |    |    |    |    |    |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |' ' |'$' |'f' |' ' |    |    |    |    |    |    |    |    |    |    |    |    |

### Byte Index `36`

By searching `(36` in the file, we found this condition: `uint8(36) + 4 == 72`, which directly identifies the value `uint8(36) = 68 ('D')`. This updates our flag to: `1RuleADayK33p$M?lw4r3Aw4y@flare-on.com`. At this point, we can guess the last missing byte could be `'a'`, `'A'`, `'4'`, or `'@'`. It’s tempting to make a final guess, but since we’re so close, let’s try to identify the last byte.

### Byte Index `45` - The Very Last Piece

Among multiple noisy conditions (which are always true), we found `uint8(45) ^ 9 == 104`, which deduces the value as `uint8(45) = 104 ^ 9 = 97 ('a')`.

### THE FINAL FLAG
| Index | 0  | 1  | 2  | 3  | 4  | 5  | 6  | 7  | 8  | 9  | 10 | 11 | 12 | 13 | 14 | 15 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |    |    |    |'e' |' ' |'f' |'l' |    |    |    |'o' |'n' |' ' |'{' |    |    |

| Index | 16 | 17 | 18 | 19 | 20 | 21 | 22 | 23 | 24 | 25 | 26 | 27 | 28 | 29 | 30 | 31 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |    |'r' |'i' |'n' |'g' |    |':' |' ' |'$' |'f' |    |    |' ' |'"' |'1' |'R' |

| Index | 32 | 33 | 34 | 35 | 36 | 37 | 38 | 39 | 40 | 41 | 42 | 43 | 44 | 45 | 46 | 47 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |'u' |'l' |'e' |'A' |'D' |'a' |'y' |'K' |'3' |'3' |'p' |'$' |'M' |'a' |'l' |'w' |

| Index | 48 | 49 | 50 | 51 | 52 | 53 | 54 | 55 | 56 | 57 | 58 | 59 | 60 | 61 | 62 | 63 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |'4' |'r' |'3' |'A' |'w' |'4' |'y' |'@' |'f' |'l' |'a' |'r' |'e' |'-' |'o' |'n' |

| Index | 64 | 65 | 66 | 67 | 68 | 69 | 70 | 71 | 72 | 73 | 74 | 75 | 76 | 77 | 78 | 79 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |'.' |'c' |'o' |'m' |'"' |' ' |'c' |'o' |'n' |'d' |    |    |    |    |    |    |

| Index | 80 | 81 | 82 | 83 | 84 |    |    |    |    |    |    |    |    |    |    |    |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |' ' |'$' |'f' |' ' |    |    |    |    |    |    |    |    |    |    |    |    |

We have completely reveal the FLLLLLAAAAAGGGGG: `1RuleADayK33p$Malw4r3Aw4y@flare-on.com`

### The Final Pokédex

Let’s continue filling in the table with the remaining conditions to complete the analysis.

| Index | 0  | 1  | 2  | 3  | 4  | 5  | 6  | 7  | 8  | 9  | 10 | 11 | 12 | 13 | 14 | 15 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |'r' |'u' |'l' |'e' |' ' |'f' |'l' |'a' |'r' |'e' |'o' |'n' |' ' |'{' |' ' |'s' |

| Index | 16 | 17 | 18 | 19 | 20 | 21 | 22 | 23 | 24 | 25 | 26 | 27 | 28 | 29 | 30 | 31 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |'t' |'r' |'i' |'n' |'g' |'s' |':' |' ' |'$' |'f' |' ' |'=' |' ' |'"' |'1' |'R' |

| Index | 32 | 33 | 34 | 35 | 36 | 37 | 38 | 39 | 40 | 41 | 42 | 43 | 44 | 45 | 46 | 47 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |'u' |'l' |'e' |'A' |'D' |'a' |'y' |'K' |'3' |'3' |'p' |'$' |'M' |'a' |'l' |'w' |

| Index | 48 | 49 | 50 | 51 | 52 | 53 | 54 | 55 | 56 | 57 | 58 | 59 | 60 | 61 | 62 | 63 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |'4' |'r' |'3' |'A' |'w' |'4' |'y' |'@' |'f' |'l' |'a' |'r' |'e' |'-' |'o' |'n' |

| Index | 64 | 65 | 66 | 67 | 68 | 69 | 70 | 71 | 72 | 73 | 74 | 75 | 76 | 77 | 78 | 79 |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |'.' |'c' |'o' |'m' |'"' |' ' |'c' |'o' |'n' |'d' |'i' |'t' |'i' |'o' |'n' |':' |

| Index | 80 | 81 | 82 | 83 | 84 |    |    |    |    |    |    |    |    |    |    |    |
|-------|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Value |' ' |'$' |'f' |' ' |'}' |    |    |    |    |    |    |    |    |    |    |    |

To make it simpler, here is the final string that matches all conditions: `rule flareon { strings: $f = "1RuleADayK33p$Malw4r3Aw4y@flare-on.com" condition: $f }`

If we hash this final string using MD5, we get the digest `b7dc94ca98aa58dabb5404541c812db2`, which matches the description and condition in the file: `hash.md5(0, filesize) == "b7dc94ca98aa58dabb5404541c812db2"`.
```bash
$ python3
Python 3.13.0 (main, Oct  7 2024, 05:02:14) [Clang 15.0.0 (clang-1500.3.9.4)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> import hashlib
>>> digest = hashlib.md5('rule flareon { strings: $f = "1RuleADayK33p$Malw4r3Aw4y@flare-on.com" condition: $f }'.\
encode('utf-8')).hexdigest()
>>> print(digest)
b7dc94ca98aa58dabb5404541c812db2
```

## Conclusion

Given the `aray.yara` file with tons of rules, we observed a pattern: only a few rules were helpful, while the others served as noise to distract from reversing attempts (which explains why only 85 bytes required 500+ rules). Ultimately, we recovered the Yara rule containing the flag, making for a very satisfying puzzle to solve! This was a new type of challenge, and I loved solving it during the competition.
