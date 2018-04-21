[Source](https://mariadb.com/kb/en/library/compound-composite-indexes/ "Permalink to Compound (Composite) Indexes - MariaDB Knowledge Base")

# Compound (Composite) Indexes - Kiến thức cơ bản về MariaDB

## Một bài học nhỏ trong "compound indexes" ("composite indexes")

Tài liệu này ban đầu có vẻ đơn giản và nhàm chán, nhưng đã được xây dựng thông tin thú vị hơn, có lẽ những điều bạn không nhận ra về cách MariaDB và chỉ mục MySQL hoạt động.

Điều này cũng giải thích [EXPLAIN][1] (ở một mức độ nào đó).

(Hầu hết điều này cũng áp dụng cho các cơ sở dữ liệu không phải MySQL.)

## The query to discuss

Câu hỏi đặt ra là "Andrew Johnson là tổng thống Hoa Kỳ khi nào?".

Bảng `Presidents` có sẵn trông giống như sau:
    
    
    +-----+------------+----------------+-----------+
    | seq | last_name  | first_name     | term      |
    +-----+------------+----------------+-----------+
    |   1 | Washington | George         | 1789-1797 |
    |   2 | Adams      | John           | 1797-1801 |
    ...
    |   7 | Jackson    | Andrew         | 1829-1837 |
    ...
    |  17 | Johnson    | Andrew         | 1865-1869 |
    ...
    |  36 | Johnson    | Lyndon B.      | 1963-1969 |
    ...
    

("Andrew Johnson" đã được chọn cho bài học này vì nó lặp lại)

Chỉ số nào sẽ là tốt nhất cho câu hỏi đó? Cụ thể hơn, điều gì tốt nhất cho
    
    
        SELECT  term
            FROM  Presidents
            WHERE  last_name = 'Johnson'
              AND  first_name = 'Andrew';
    

Một số INDEX để thử ...

* Không có chỉ mục nào
* INDEX(first_name), INDEX(last_name) (hai chỉ mục riêng biệt) 
* "Index Merge Intersect" 
* INDEX(last_name, first_name) ( một chỉ mục "compound") 
* INDEX(last_name, first_name, term) ( một chỉ mục "covering") 
* Các biến thể 

## Không có chỉ mục nào

Vâng, tôi đang fudging một chút ở đây. Tôi có một PRIMARY KEY trên `seq`, nhưng điều đó không có lợi thế về truy vấn chúng tôi đang nghiên cứu.
    
    
    mysql>  SHOW CREATE TABLE Presidents G
    CREATE TABLE `presidents` (
      `seq` tinyint(3) unsigned NOT NULL AUTO_INCREMENT,
      `last_name` varchar(30) NOT NULL,
      `first_name` varchar(30) NOT NULL,
      `term` varchar(9) NOT NULL,
      PRIMARY KEY (`seq`)
    ) ENGINE=InnoDB AUTO_INCREMENT=45 DEFAULT CHARSET=utf8
    
    mysql>  EXPLAIN  SELECT  term
                FROM  Presidents
                WHERE  last_name = 'Johnson'
                  AND  first_name = 'Andrew';
    +----+-------------+------------+------+---------------+------+---------+------+------+-------------+
    | id | select_type | table      | type | possible_keys | key  | key_len | ref  | rows | Extra       |
    +----+-------------+------------+------+---------------+------+---------+------+------+-------------+
    |  1 | SIMPLE      | Presidents | ALL  | NULL          | NULL | NULL    | NULL |   44 | Using where |
    +----+-------------+------------+------+---------------+------+---------+------+------+-------------+
    
    # Hoặc,sử dụng hình thức hiển thị khác:  EXPLAIN ... G
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: ALL        <-- Implies table scan
    possible_keys: NULL
              key: NULL       <-- Implies that no index is useful, hence table scan
          key_len: NULL
              ref: NULL
             rows: 44         <-- That's about how many rows in the table, so table scan
            Extra: Using where
    

## Chi tiết thực hiện

Trước tiên, hãy mô tả cách InnoDB lưu trữ và sử dụng các chỉ mục.

* Dữ liệu và PRIMARY KEY được "clustered" lại với nhau trên BTree. 
* Tra cứu BTree khá nhanh và hiệu quả. Đối với một bảng hàng triệu dòng có thể có 3 cấp độ của BTree, và hai cấp cao nhất có thể được lưu trữ. 
* Mỗi chỉ số phụ nằm trong một BTree khác, với PRIMARY KEY ở lá. 
* Tìm nạp các mục 'consecutive' (theo chỉ mục) từ BTree rất hiệu quả vì chúng được lưu trữ liên tiếp.
* Để đơn giản, chúng ta có thể đếm từng tra cứu BTree dưới dạng 1 đơn vị công việc và bỏ qua các lần quét cho các mục liên tiếp. Điều này xấp xỉ số lần truy cập đĩa cho một bảng lớn trong một hệ thống bận. 

Đối với MyISAM, PRIMARY KEY không được lưu trữ với dữ liệu, vì vậy hãy nghĩ về nó như là một khóa thứ cấp (rất đơn giản).

## INDEX(first_name), INDEX(last_name)

Với người mới làm quen, khi anh ấy biết về lập chỉ mục, hãy quyết định lập chỉ mục nhiều cột, mỗi lần một cột. Nhưng...

MySQL hiếm khi sử dụng nhiều hơn một chỉ mục tại một thời điểm trong một truy vấn. Vì vậy, nó sẽ phân tích các chỉ số có thể.

* first_name -- có 2 dòng khả dụng (một tra cứu BTree, say đó duyệt lần lượt) 
* last_name -- có 2 dòng khả dụng.Giả sử nó chọn last_name. Dưới đây là các bước để thực hiện SELECT: 1\. Sử dụng INDEX(last_name), tìm 2 mục chỉ mục với last_name = 'Johnson'. 2\. Lấy được PRIMARY KEY (ngầm được thêm vào mỗi chỉ số phụ trong InnoDB); lấy (17, 36). 3\. Tiếp cận dữ liệu bằng cách sử dụng seq = (17, 36) để có được các dòng cho Andrew Johnson and Lyndon B. Johnson. 4\. Sử dụng phần còn lại của mệnh đề WHERE lọc tất cả trừ hàng mong muốn. 5\. Đưa ra kết quả (1865-1869). 
    
    
    mysql>  EXPLAIN  SELECT  term
                FROM  Presidents
                WHERE  last_name = 'Johnson'
                  AND  first_name = 'Andrew'  G
      select_type: SIMPLE
            table: Presidents
             type: ref
    possible_keys: last_name, first_name
              key: last_name
          key_len: 92                 <-- VARCHAR(30) utf8 may need 2+3*30 bytes
              ref: const
             rows: 2                  <-- Two 'Johnson's
            Extra: Using where
    

## "Index Merge Intersect"

OK, vì bạn thực sự thông minh và quyết định rằng MySQL phải đủ thông minh để sử dụng cả hai chỉ mục tên để lấy kết quả. Điều này được gọi là "Intersect". 1\. Sử dụng INDEX(last_name), tìm 2 mục chỉ mục với last_name = 'Johnson'; được (7, 17) 2\. Sử dụng INDEX(first_name), tìm 2 mục chỉ mục với first_name = 'Andrew'; được (17, 36) 3\. "And" hai danh sách cùng nhau (7,17) & (17,36) = (17) 4\. Tiếp cận dữ liệu bằng cách sử dụng seq = (17) để có được hàng cho Andrew Johnson. 5\. Đưa ra kết quả (1865-1869).
    
    
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: index_merge
    possible_keys: first_name,last_name
              key: first_name,last_name
          key_len: 92,92
              ref: NULL
             rows: 1
            Extra: Using intersect(first_name,last_name); Using where
    

EXPLAIN không cung cấp thông tin chi tiết về số lượng hàng được thu thập từ mỗi chỉ mục, v.v.

## INDEX(last_name, first_name)

Đây được gọi là chỉ mục "compound" hoặc "composite" vì nó có nhiều cột. 1\. Đi sâu vào chỉ mục trong BTree có được chính xác dòng chỉ mục cho Johnson + Andrew; được seq = (17). 2\. Tiếp cận dữ liệu bằng cách sử dụng seq = (17) để có được hàng Andrew Johnson. 3\. Đưa ra kết quả (1865-1869). Điều này tốt hơn nhiều. Trong thực tế, điều này luôn là "best".
    
    
        ALTER TABLE Presidents
            (drop old indexes and...)
            ADD INDEX compound(last_name, first_name);
    
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: ref
    possible_keys: compound
              key: compound
          key_len: 184             <-- Độ dài của cả hai trường
              ref: const,const     <-- Mệnh đề WHERE cho các hằng số cho cả hai
             rows: 1               <-- Goodie!  It homed in on the one row.
            Extra: Using where
    

## "Covering": INDEX(last_name, first_name, term)

Surprise! Chúng tôi thực sự có thể làm tốt hơn một chút. Chỉ mục "Covering" là chỉ mục trong đó _tất cả_ các trường của SELECT được tìm thấy trong chỉ mục. Nó có thêm điểm cộng không phải tiếp cận "data" để hoàn thành công việc. 1\. Đi sâu vào BTree để có được chỉ mục chính xác của hàng Johnson+Andrew; được seq = (17). 2\. Đưa ra kết quả (1865-1869). "data" của BTree không được động tới; đây là một cải tiến so với "compound".
    
    
        ... ADD INDEX covering(last_name, first_name, term);
    
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: ref
    possible_keys: covering
              key: covering
          key_len: 184
              ref: const,const
             rows: 1
            Extra: Using where; Using index   <-- Note
    

Mọi thứ đều tương tự như sử dụng  "compound", ngoại trừ việc bổ sung "Using index".

## Các biến thể

* Điều gì sẽ xảy ra nếu bạn xáo trộn các trường trong mệnh đề WHERE? Trả lời: Thứ tự của những thứ trong AND không quan trọng. 
* Điều gì sẽ xảy ra nếu bạn xáo trộn các trường trong INDEX? Trả lời: Nó có thể tạo nên sự khác biệt lớn. Xem thêm sau một phút nữa. 
* Điều gì sẽ xảy ra nếu có thêm trường vào cuối? Trả lời: Có 1 chút ảnh hưởng; có thể rất nhiều (ví dụ, 'covering'). 
* Dư thừa? Sẽ thế nào, nếu bạn có cả hai: INDEX(a), INDEX(a,b)? Trả lời: Dư thừa chi phí một cái gì đó trong INSERT; nó hiếm khi hữu ích cho các SELECT. 
* Tiền tố? Đó là, INDEX(last_name(5). first_name(5)) Trả lời : Đừng bận tâm; nó hiếm khi có tác dụng, và thường là làm hại. (Chi tiết trong một topic khác) 

## Thêm ví dụ:
    
    
        INDEX(last, first)
        ... WHERE last = '...' -- tốt (mặc dù `first` không được sử dụng)
        ... WHERE first = '...' -- chỉ mục là vô ích
    
        INDEX(first, last), INDEX(last, first)
        ... WHERE first = '...' -- Chỉ mục thứ nhất được sử dụng
        ... WHERE last = '...' -- Chỉ mục thứ hai được sử dụng
        ... WHERE first = '...' AND last = '...' -- hay có thể được sử dụng tốt như nhau
    
        INDEX(last, first)
        Cả hai được xử lý bởi một INDEX:
        ... WHERE last = '...'
        ... WHERE last = '...' AND first = '...'
    
        INDEX(last), INDEX(last, first)
        Trong ví dụ trên, đừng bận tâm INDEX(last).
    

## Postlog

Refreshed -- Oct, 2012; more links -- Nov 2016