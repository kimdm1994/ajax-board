# AJAX로 게시판 만들기

+ 메인페이지 (C/R/U/D)
+ 서브쿼리 (freeBoardGetList)
+ Controller (try ~ catch) 활용

> Main Controller
```java
@RequestMapping(value = "/main.ino", method = RequestMethod.GET)
public ModelAndView main(HttpServletRequest request,
		@RequestParam(value = "page", defaultValue = "1", required = false) int page,
		@RequestParam(value = "listLimit", defaultValue = "10", required = false) int listLimit
		) {
	ModelAndView model = new ModelAndView();
	Map<String, Object> map = new HashMap<>();
	List<Map<String, Object>> codeName = null;
	List<FreeBoardDto> list = null;
	PageInfo pageInfo = null;
	int listCount = 0;
	int pageLimit = 5;

	codeName = freeBoardService.searchGroup();
	model.addObject("codeName", codeName);

	listCount = freeBoardService.freeBoardCount(map);
	// pageInfo = new PageInfo(현재페이지, 보여질페이지번호개수, 전체리스트개수, 한페이지의표시될리스트수);
	pageInfo = new PageInfo(page, pageLimit, listCount, listLimit);

	map.put("listCount", listCount);
	map.put("pageInfo", pageInfo);
	list = freeBoardService.freeBoardList(map);

	model.addObject("freeBoardList", list);
	model.addObject("pageInfo", pageInfo);
	model.setViewName("boardMain");
	return model;
}

@ResponseBody
@RequestMapping("/list.ino")
public Map<String, Object> boardList(HttpServletRequest request,
    @RequestParam(value = "keyword", defaultValue = "", required = false) String keyword,
    @RequestParam(value = "searchType", defaultValue = "0", required = false) String searchType,
    @RequestParam(value = "startDate", defaultValue = "", required = false) String startDate,
    @RequestParam(value = "endDate", defaultValue = "", required = false) String endDate,
    @RequestParam(value = "page") int page
    ) {
  Map<String, Object> map = new HashMap<>();
  List<FreeBoardDto> list = null;
  PageInfo pageInfo = null;

  int listCount = 0;
  int pageLimit = 5;
  if (searchType != null || keyword != null || startDate != null || endDate != null) {
    searchType = searchType.trim();
    keyword = keyword.trim();
    startDate = startDate.trim();
    endDate = endDate.trim();
  }

  map.put("searchType", searchType);
  map.put("keyword", keyword);
  map.put("startDate", startDate);
  // pageInfo = new PageInfo(현재페이지, 보여질페이지번호개수, 전체리스트개수, 한페이지의표시될리스트수);
  listCount = freeBoardService.freeBoardCount(map);
  pageInfo = new PageInfo(page, 5, listCount, 10);


  map.put("listCount", listCount);
  map.put("pageInfo", pageInfo);

  list = freeBoardService.freeBoardList(map);
  map.put("list", list);
  System.out.println(pageInfo);
  return map;
}

// Write Page
@RequestMapping("/freeBoardInsert.ino")
public String freeBoardInsert(HttpServletRequest request, FreeBoardDto dto) {

  return "freeBoardInsert";
}

// Update Page
@RequestMapping("/freeBoardInsertPro.ino")
@ResponseBody
public Map<String, Object> freeBoardInsertPro(HttpServletRequest request, @RequestParam Map<String, Object> map) {
  String result = "";
  try {
    // 성공 시 객체 넘기고 MAX(NUM) 값을 넣어준다,
    // Key : NUM, Value : MAX(NUM)
    freeBoardService.freeBoardInsertPro(map);
    int maxNum = freeBoardService.getNewNum();
    result = "true";
    map.put("maxNum", maxNum);
    map.put("result", result);
  } catch (Exception e) {
    // 실패 시 실패 익셉션 오류 넘겨주고
    map.put("message", e.getMessage());
    result = "false";
    map.put("result", result);
    return map;
  }
  return map;
}

// Detail Page
@RequestMapping("/freeBoardDetail.ino")
public ModelAndView freeBoardDetail(ModelAndView model, HttpServletRequest request, FreeBoardDto freeBoardDto,
    @RequestParam("num") int num) {
  freeBoardDto = freeBoardService.findNameByNum(num);

  model.addObject("freeBoardDto", freeBoardDto);
  model.setViewName("freeBoardDetail");

  return model;
  // return new ModelAndView("freeBoardDetail", "freeBoardDto", null);
}
```

> Main Service
```java
public int freeBoardCount(Map<String, Object> map) {
	return sqlSessionTemplate.selectOne("freeBoardCount", map);
}

public List<FreeBoardDto> freeBoardList(Map<String, Object> map) {
  return sqlSessionTemplate.selectList("freeBoardGetList", map);
}
```

> Main Mapper
```xml
<mapper namespace="ino.web.freeBoard.mapper.FreeBoardMapper">
<!-- 
	1. #{value} 에는 따음표 쓰지말기.. 
	2. searchType 넘길 때 int형으로 넘어와서 .toString() 해줘야 한다. 
	3. SubQuery는 하나의 값만 리턴한다. (freeBoardGetList 참고, 매번 추가/삭제 부분을 서브쿼리를 사용함으로서 유지보수에 도움을 준다.)
-->
<sql id="search">
	<if test="searchType != null">
		<if test="searchType == '1'.toString()"> <!-- 타입 -->
			AND CODE_TYPE LIKE '%'||#{keyword}||'%'
		</if> <!-- 제목 -->
		<if test="searchType == '2'.toString()">
			AND TITLE LIKE '%'||#{keyword}||'%'
		</if> <!-- 내용 -->
		<if test="searchType == '3'.toString()">
			AND CONTENT LIKE '%'||#{keyword}||'%'
		</if> <!-- 번호는 정확히 입력해야만? -->
		<if test="searchType == '4'.toString()">
			AND NUM LIKE #{keyword}
		</if> <!-- 글쓴이 -->
		<if test="searchType == '5'.toString()">
			AND NAME LIKE '%'||#{keyword}||'%'
		</if> <!-- TO_CHAR : java.sql.SQLException: ORA-01481: invalid number format model, REGDATE 는 DATE 형식이기 때문에 CHAR로 바꾸어주어야 한다. -->
		<if test="searchType == '6'.toString()">
			AND TO_CHAR(REGDATE, 'YYYY/MM/DD') BETWEEN TO_CHAR(#{startDate}, 'YYYY/MM/DD') AND TO_CHAR(#{endDate}, 'YYYY/MM/DD')
		</if>
	</if>
</sql>

<select id="searchGroup" resultType="Map">
	select d.code_name,d.code
	from codem m
	inner join coded d on m.gr_code = d.gr_code and m.gr_code='GR002'
</select>

<!-- resultType="ino.web.board.dto.BoardDto" -->
<select id="freeBoardGetList" resultType="freeBoardDto" parameterType="map"> 
	SELECT RNUM, (Select code_name from codem a, coded b where  a.gr_code = b.gr_code and a.gr_code='GR002' and b.code = T.CODE_TYPE) as codeType, NUM, NAME, TITLE, CONTENT, TO_CHAR(REGDATE, 'YYYY/MM/DD') AS REGDATE
	FROM (SELECT ROWNUM AS RNUM, S.*
	      FROM FREEBOARD S
	  where 1=1
	      <include refid="search" />
	      ORDER BY NUM ASC
	      ) T
	WHERE RNUM BETWEEN (#{pageInfo.currentPage} * #{pageInfo.listLimit} - (#{pageInfo.listLimit} - 1)) AND (#{pageInfo.currentPage} * #{pageInfo.listLimit})
</select>

<insert id="freeBoardInsertPro" parameterType="Map">
	INSERT INTO FREEBOARD(CODE_TYPE, NUM, TITLE, NAME, REGDATE, CONTENT)
	VALUES( #{codeType}, SEQ_NUM.NEXTVAL, #{title}, #{name}, SYSDATE, #{content})
</insert>

<select id="freeBoardDetailByNum" resultType="freeBoardDto" parameterType="int">
	SELECT CODE_TYPE, NUM, TITLE, NAME, TO_CHAR(REGDATE,'YYYY/MM/DD') REGDATE, CONTENT FROM FREEBOARD
	WHERE NUM=#{num}
</select>

<select id="freeBoardNewNum" resultType="int">
	SELECT MAX(NUM)
	FROM FREEBOARD
</select>

<select id="freeBoardCount" resultType="int"  parameterType="map">
	SELECT COUNT(*)
	FROM FREEBOARD
	where 1=1
	<include refid="search" />
</select>

<update id="freeBoardModify" parameterType="map">
	UPDATE FREEBOARD
	SET CODE_TYPE = #{codeType}, TITLE = #{title}, CONTENT = #{content}
	WHERE NUM = #{num}
</update>

<update id="freeBoardDelete" parameterType="int">
	DELETE FROM FREEBOARD
	WHERE NUM = #{num}
</update>

<!-- Mybatis 내에서 forEach문으로 처리 -->
<update id="deleteBoardTest" parameterType="map">
	DELETE FROM FREEBOARD
	WHERE NUM IN
		<foreach collection="arrayValue" item="num" open="(" separator="," close=")">
			#{num}
		</foreach>
</update>

<select id="findNameByNum" parameterType="int" resultType="freeBoardDto">
	SELECT CODE_TYPE as codeType, NAME, NUM, TITLE, CONTENT FROM FREEBOARD 
	WHERE NUM = #{num}
</select>
</mapper>
```

> Main View
```javascript
<script type="text/javascript">
  $(function() {
    var deleteBoxAll = $('input[name="deleteBoxAll"]');
    var deleteBox = $('input[name="deleteBox"]');
    var rowCnt = deleteBox.length;

    // 전체 체크박스 클릭
    deleteBoxAll.on('click', function() {
      for (var i = 0 ; i < deleteBox.length ; i++) {
        deleteBox[i].checked = this.checked;
      }
    });

    // 전체 체크박스가 선택되어있는데, 체크박스가 헤제되면
    deleteBox.on('click', function() {
      if ($('input[name="deleteBox"]:checked').length == rowCnt) {
        deleteBoxAll[0].checked = true;
      } else {
        deleteBoxAll[0].checked = false;
      }
    });

    var deleteButton = $('#deleteButton');
    deleteButton.on('click', function() {
      var arrayValue = new Array(); // 체크 값 저장할 배열
      var list = $('input[name="deleteBox"]');
      for (var i = 0 ; i < list.length ; i++) {
        if (list[i].checked) {
          arrayValue.push(list[i].value);
        }
      }
      if (arrayValue.length == 0) {
        alert('선택된 게시글이 없습니다.');
      } else {
        var conf = confirm('정말 삭제하시겠습니까?');
        if (conf) {
          $.ajax({
            url: './freeBoardDelete.ino',
            type: 'post',
            traditional: true, // 배열로 전송
            data: {
              arrayValue: arrayValue
            },
            success: function(data) {
              if (data.result) {
                alert('성공적으로 삭제되었습니다.');
                location.replace("./main.ino"); // 새로고침
              } else {
                alert('삭제를 실패하였습니다.');
                return;
              }
            },
            error: function(e) {
              alert('삭제하는데 오류가 발생했습니다.');
            }
          });
        }
      }
    });

    // 검색관련 SelectBox
    var selected = $('#searchType');
    selected.change(function () {
      if (selected.val() == 0) {
        if ($('span[name=enterInput]').children().remove());
      }
      if (selected.val() == 1) {
        if ($('span[name=enterInput]').children().remove()) {
          $('span[name=enterInput]').append('<select name="keyword" id="keyword"><option value="01">' 
              + '자유' + '</option><option value="02">' + '익명' + '</option><option value="03">' + 'QnA' + '</option></select>')
        }
      }
      if (selected.val() == 2) {
        if ($('span[name=enterInput]').children().remove()) {
          $('span[name=enterInput]').append('<input type=text name="keyword" id="keyword" placeholder="글제목">');
        }
      }
      if (selected.val() == 3) {
        if ($('span[name=enterInput]').children().remove()) {
          $('span[name=enterInput]').append('<input type=text name="keyword" id="keyword" placeholder="글내용">');
        }
      }
      if (selected.val() == 4) {
        if ($('span[name=enterInput]').children().remove()) {
          $('span[name=enterInput]').append('<input type=text name="keyword" id="searchNum" placeholder="글번호">');
        }
      }
      if (selected.val() == 5) {
        if ($('span[name=enterInput]').children().remove()) {
          $('span[name=enterInput]').append('<input type=text name="keyword" id="keyword" placeholder="글쓴이">');
        }
      }
      if (selected.val() == 6) {
        if ($('span[name=enterInput]').children().remove()) {
          $('span[name=enterInput]').append('<input type=text name="startDate" id="startDate" placeholder="시작"> <input type=text name="endDate" id="endDate" placeholder="종료">');
        }
      }

      // 숫자만 입력 regExp
      $('input[id="searchNum"]').on('input', function() {
        this.value = this.value.replace(/[^0-9.]/g, '').replace(/(\..*)\./g, '$1');
      });
    });
  });
  function search(page){
    console.log(1);
    var keyword = $('#keyword').val();
    var searchType = $('#searchType').val(); 
    var startDate = $('#startDate').val(); 
    var endDate = $('#endDate').val(); 
    var page = page;
    // 빈 값 처리
    if(keyword == "" || startDate == "" || endDate == "") {
      alert('값을 입력해주세요');
      $('input[name=keyword]').focus();
      $('#startDate').focus();
      $('#endDate').focus();
      return false;
    } else {
      $.ajax({
        url: "./list.ino",
        data: {
          keyword: keyword,
          searchType: searchType,
          startDate: startDate,
          endDate: endDate,
          page: page
        },
        success: function(data) {
          //console.log(1)
          $('#tb').html("");
          $('#pageBar').html("");
          var str = "";
          var str2 = "";
          var list = data.list;
          var pageInfo = data.pageInfo;
          console.log(pageInfo);
          var currentPage = data.pageInfo.currentPage;
          var startPage = data.pageInfo.startPage;
          var maxPage = data.pageInfo.maxPage;
          var endPage = data.pageInfo.endPage;
          var listLimit = data.pageInfo.listLimit;
            $.each(list, function(i) {
              str += '<tr id="rowBoardList" name="rowBoardList"><td style="width: 5px;"><input name="deleteBox" type="checkbox" value="' 
                  + list[i].num + '"/></td><td style="width: 55px; padding-left: 30px;" align="center">' 
                  + list[i].codeType + '</td><td style="width: 50px; padding-left: 10px;" align="center">' 
                  + list[i].num + '</td><td style="width: 125px; align="center"><a href="./freeBoardDetail.ino?num=' + list[i].num + '">' 
                  + list[i].title + '</a></td><td style="width: 48px; padding-left: 50px;" align="center">' 
                  + list[i].name + '</td><td style="width: 100px; padding-left: 95px;" align="center">' 
                  + list[i].regdate + '</td><tr>';
            });
          $('#tb').append(str);
            for (var a = startPage; a <= endPage; a++) {
              str2 += '<a href=javascript:search('+a+')>' + a + '</a>';
            }
            console.log(str2);
          $('#pageBar').append(str2);
        },
        error: function(e) {
          console.log(e);
        }
      });
    }
  }
</script>
```
> Insert View
```javascript
<script type="text/javascript">
// null
$(document).ready(function() {
    $('#submitBtn').on('click', function() {
        var codeType = $('#selectedOption').val();
        var name = $('#name').val();
        var content = $('#content').val();
        var title = $('#title').val();

    if (name == "" || name.length == 0) {
        alert('이름을 입력해주세요!');
        $('#name').focus();
        return false;
    } else if (title == "" || title.length == 0) {
        alert('이름을 입력해주세요!');
        $('#title').focus();
        return false;
    } else {
    // cf 조건문 수행
    var cf = confirm('작성하시겠습니까?');
    if (cf) {
        $.ajax({
            url: "./freeBoardInsertPro.ino",
            type: 'post',
            data: {
                codeType: codeType,
                name: name,
                title: title,
                content: content
            },
            success: function(data) {
                if (data.result) {
                    alert("게시글이 등록되었습니다.");
                    var agree = confirm('메인페이지로 이동하시겠습니까?');
                if (agree) {
                    location.href = "./main.ino";
                } else {
                    location.href = "./freeBoardDetail.ino?num=" + data.maxNum;
                }
        } else {
                // Exception 발생 시
                alert('게시글 등록을 실패하였습니다.');
                var dataMessage = data.message;
                $('#printMessage').html(dataMessage);
            }
        },
            error: function(e) {
                console.log(e);
                    }
                });
            }
        }; 
    });
});
</script>
```
> Delete View
```javascript
<script type="text/javascript">
// AJAX로 게시글 수정, 삭제
$(function() {
    $('#modifyDetail').on('click', function() {
            var codeType = $('#codeType').val();
            var num = $('#num').val();
            var content = $('#content').val();
            var title = $('#title').val();

            // cf 조건문 수행
            var cf = confirm('수정하시겠습니까?');
            if (cf) {
                $.ajax({
                    url: "./freeBoardModify.ino",
                    type: 'post',
                    data: {
                        codeType: codeType,
			num: num,
			title: title,
			content: content
			},
		    success: function(data) {
			if (data.result) {
			    // try 성공
			    alert('게시글 수정을 성공하였습니다.');
			    var agree = confirm('메인페이지로 이동하시겠습니까?');
			    if (agree) {
			        location.href = "./main.ino";	
			    } else {
			        location.href = "./freeBoardDetail.ino?num=" + data.num;
			    }
			} else {
			    // Exception 발생 시 
			    alert('게시글 수정을 실패하였습니다.');
			    dataMessage = data.message;
			    $('#printMessage').html(dataMessage);
			    }
			},
			error: function(e) {
			       console.log(e);
                }
            });
        }
    });
});
</script>
```
