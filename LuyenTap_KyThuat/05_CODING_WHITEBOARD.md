# 5. Coding / Whiteboard

Vòng giải thuật thường kiểm: bạn **làm rõ đề**, **nói to cách nghĩ**, **viết code chạy được**, **bàn độ phức tạp**. Với fresher/junior phần lớn ở mức Easy–Medium.

## Quy trình UMPIRE (nói to từng bước)
1. **U**nderstand: hỏi làm rõ (input/output, ràng buộc, ca biên: rỗng, âm, trùng, rất lớn).
2. **M**atch: bài này giống dạng nào (mảng/chuỗi/hash/2 con trỏ/đệ quy...).
3. **P**lan: phác hướng giải bằng lời/giả mã trước khi gõ.
4. **I**mplement: viết, vừa viết vừa giải thích.
5. **R**eview: chạy thử với 1–2 ví dụ (kể cả ca biên).
6. **E**valuate: độ phức tạp thời gian/không gian, có thể tối ưu không.

## Big-O cần nhớ
- O(1) hằng · O(log n) nhị phân · O(n) duyệt · O(n log n) sort · O(n²) hai vòng lồng.
- Dùng **hash map/set** để đổi O(n²) → O(n) (tra thành viên O(1)).

## Các dạng hay gặp (Easy–Medium)
- **Chuỗi**: đảo chuỗi, kiểm palindrome, đếm ký tự, **anagram** (so Counter). *(Liên hệ: word-checker dùng regex/đếm tần suất.)*
- **Mảng**: two-sum (hash map), max/min, gộp khoảng, di chuyển số 0.
- **Hai con trỏ / cửa sổ trượt**: dãy con tổng lớn nhất, chuỗi con không lặp ký tự.
- **Hash**: đếm tần suất, phần tử xuất hiện nhiều nhất (vd `Counter.most_common`).
- **Đệ quy / duyệt**: giai thừa, Fibonacci (có memo), duyệt cây/đồ thị (DFS/BFS).
- **Sắp xếp + tìm kiếm**: binary search trên mảng đã sort.

## Ví dụ mẫu (Python — viết được trên bảng)

**Two-sum (O(n)):**
```python
def two_sum(nums, target):
    seen = {}                 # value -> index
    for i, x in enumerate(nums):
        if target - x in seen:
            return [seen[target - x], i]
        seen[x] = i
    return None
```

**Đếm tần suất / phần tử nhiều nhất:**
```python
from collections import Counter
def most_common_word(words):
    return Counter(words).most_common(1)[0][0]
```
*(Chính là ý tưởng word-checker chọn font phổ biến nhất, quan-quen-map gợi ý có trọng số.)*

**Kiểm palindrome (hai con trỏ):**
```python
def is_palindrome(s):
    i, j = 0, len(s) - 1
    while i < j:
        if s[i] != s[j]:
            return False
        i += 1; j -= 1
    return True
```

## Mẹo whiteboard
- **Đừng im lặng gõ**: nói cách nghĩ — họ chấm tư duy.
- **Hỏi ca biên trước**: rỗng, một phần tử, trùng, tràn số.
- Bí thì nói brute-force trước rồi tối ưu dần ("em làm O(n²) trước, rồi dùng hash về O(n)").
- Viết tên biến rõ ràng; kiểm thử bằng tay với ví dụ nhỏ.

## Câu hỏi mẫu
- "Độ phức tạp của giải pháp em?" → nêu time + space.
- "Tối ưu hơn được không?" → đổi cấu trúc dữ liệu (mảng→hash), giảm vòng lặp.
- "Xử lý input rất lớn?" → tránh O(n²), cân nhắc xử lý theo luồng.

> 💡 Luyện ~20–30 bài Easy/Medium trên LeetCode/HackerRank theo dạng (chuỗi, mảng, hash, hai con trỏ) là đủ tự tin cho vòng coding fresher/junior.
