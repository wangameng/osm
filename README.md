osm
===

osm (Object Sql Mapping And Template) is an ORM tool written in Go. It is currently used in production environments and only supports mysql and postgresql (other databases have not been tested).

In the past, the Java server was developed using MyBatis. Its sql mapping is very flexible. It separates sql, and the program completes all database operations through input and output.

osm is a simple imitation of MyBatis. Of course, dynamic sql is generated using go and template packages, so the format of sql mapping is different from MyBatis. The format of sql xml is as follows:

    <?xml version="1.0" encoding="utf-8"?>
    <osm>
     <select id="selectUsers" result="structs">
       SELECT id,email
       FROM user
       {{if ne .Email ""}} where email=#{Email} {{end}}
       order by id
     </select>
    </osm>


## get osm

    go get github.com/yinshuwei/osm

## api doc

http://godoc.org/github.com/yinshuwei/osm

## Quickstart

Create database
    
    create database test;
    use test;

Create `user` table
    
    CREATE TABLE `user` (
      `id` int(11) NOT NULL AUTO_INCREMENT,
      `email` varchar(255) DEFAULT NULL,
      `nickname` varchar(45) DEFAULT NULL,
      `create_time` varchar(255) DEFAULT NULL,
      PRIMARY KEY (`id`)
    ) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8 COMMENT='user table';

* Execute SQL directly (Go template parsing is not supported) Example

example_sql.go
    
    package main

    import (
        "fmt"
        "time"

        _ "github.com/go-sql-driver/mysql"
        "github.com/yinshuwei/osm"
    )

    // User 用户Model
    type User struct {
        ID         int64
        Email      string
        Nickname   string
        CreateTime time.Time
    }

    func main() {
        o, err := osm.New("mysql", "root:123456@/test?charset=utf8", []string{})
        if err != nil {
            fmt.Println(err.Error())
        }

        //add
        user := User{
            Email:      "test@foxmail.com",
            Nickname:   "haha",
            CreateTime: time.Now(),
        }
        sql := "INSERT INTO user (email,nickname,create_time) VALUES (#{Email},#{Nickname},#{CreateTime});"
        fmt.Println(o.InsertBySQL(sql, user))

        //query
        user = User{
            Email: "test@foxmail.com",
        }
        var results []User
        sql = "SELECT id,email,nickname,create_time FROM user WHERE email=#{Email};"
        _, err = o.SelectStructs(sql, user)(&results)
        if err != nil {
            fmt.Println(err.Error())
        }
        for _, u := range results {
            fmt.Println(u)
        }

        //delete
        fmt.Println(o.DeleteBySQL("DELETE FROM user WHERE email=#{Email}", user))

        err = o.Close()
        if err != nil {
            fmt.Println(err.Error())
        }
    }



* Example of executing SQL in template (support go template parsing)

sql template file test.xml

    <?xml version="1.0" encoding="utf-8"?>
    <osm>
        <insert id="insertUser">
        <![CDATA[
    INSERT INTO user (email,nickname,create_time) VALUES (#{Email},#{Nickname},#{CreateTime});
        ]]>
        </insert>
    
        <select id="selectUser" result="structs">
        <![CDATA[
    SELECT id,email,nickname,create_time FROM user 
    WHERE 
    {{if ne .Email ""}}email=#{Email} and{{end}}
    {{if ne .Nickname ""}}nickname=#{Nickname} and{{end}}
    1=1;
        ]]>
        </select>
    
        <delete id="deleteUser">
        <![CDATA[
    DELETE FROM user WHERE email=#{Email}
        ]]>
        </delete>
    </osm>

example.go
    
    package main

    import (
        "fmt"
        "time"

        _ "github.com/go-sql-driver/mysql"
        "github.com/yinshuwei/osm"
    )

    // User model
    type User struct {
        ID         int64
        Email      string
        Nickname   string
        CreateTime time.Time
    }

    func main() {
        o, err := osm.New("mysql", "root:root@/test?charset=utf8", []string{"test.xml"})
        if err != nil {
            fmt.Println(err.Error())
        }

        //add
        user := User{
            Email:      "test@foxmail.com",
            Nickname:   "haha",
            CreateTime: time.Now(),
        }
        fmt.Println(o.Insert("insertUser", user))

        //query with template
        user = User{
            Email: "test@foxmail.com",
        }
        var results []User
        o.Select("selectUser", user)(&results)
        for _, u := range results {
            fmt.Println(u)
        }

        //delete
        fmt.Println(o.Delete("deleteUser", user))

        err = o.Close()
        if err != nil {
            fmt.Println(err.Error())
        }
    }



## Query result type


* value : a single line and stored in a variable of variable length (...)
    
    xml

        <select id="selectResUserValue" result="value">
            SELECT id, email, head_image_url FROM res_user WHERE email=#{Email};
        </select>

    go

        user := ResUser{Email: "test@foxmail.com"}
        var id int64
        var email, headImageURL string
        o.Select("selectResUserValue", user)(&id, &email, &headImageURL)

        log.Println(id, email, headImageURL)

* values : multiple rows and stored in variables of variable length (..., each is an array, and each array is the same length as the result set rows)
    
    xml

        <select id="selectResUserValues" result="values">
            SELECT id,email,head_image_url FROM res_user WHERE city=#{City} order by id;
        </select>

    go

        user := ResUser{City: "Shanghai"}
        var ids []int64
        var emails, headImageUrls []string
        o.Select("selectResUserValues", user)(&ids, &emails, &headImageUrls)

        log.Println(ids, emails, headImageUrls)

* struct : a single line and stored in struct
    
    xml

        <select id="selectResUser" result="struct">
            SELECT id, email, head_image_url FROM res_user WHERE email=#{Email};
        </select>

    go

        user := ResUser{Email: "test@foxmail.com"}
        var result ResUser
        o.Select("selectResUser", user)(&result)

        log.Printf("%#v", result)

* structs : multiple lines, and stored in the struct array
    
    xml

        <select id="selectResUsers" result="structs">
            SELECT id,email,head_image_url FROM res_user WHERE city=#{City} order by id;
        </select>

    go

        user := ResUser{City: "Shanghai"}
        var results []*ResUser // or var results []ResUser
        o.Select("selectResUsers", user)(&results)
        log.Printf("%#v", results)

* kvs : multiple rows, each row has two fields, the former is the key, the latter is the value, and stored in the map (two columns)
    
    xml

        <select id="selectResUserKvs" result="kvs">
            SELECT id,email FROM res_user WHERE city=#{City} order by id;
        </select>

    go

        user := ResUser{City: "Shanghai"}
        var idEmailMap map[int64]string
        o.Select("selectResUserKvs", user)(&idEmailMap)
        log.Println(idEmailMap)


## Correspondence between struct and SQL column

* Normal conversion process

    Separated by "_" (example: XXX_YYY-> XXX, YYY)

    Each part is converted to the first word uppercase and the rest of the characters lowercase (example: XXX, YYY-> Xxx, Yyy)
    
    Splicing (example: Xxx, Yyy-> XxxYyy)

* Common abbreviated words, these two words can be used in both forms, you can choose one on the struct.
    
    For example, "UserId" and "UserID" can normally correspond to the "user_id" column. However, it is not possible to have both "UserId" and "UserID" members in the same struct. If they exist at the same time, only one member will be assigned.
    
        Acl  or   ACL 
        Api  or   API 
        Ascii  or ASCII 
        Cpu  or   CPU 
        Css  or   CSS 
        Dns  or   DNS 
        Eof  or   EOF 
        Guid  or  GUID 
        Html  or  HTML 
        Http  or  HTTP 
        Https  or HTTPS 
        Id  or    ID 
        Ip  or    IP 
        Json  or  JSON 
        Lhs  or   LHS 
        Qps  or   QPS 
        Ram  or   RAM 
        Rhs  or   RHS 
        Rpc  or   RPC 
        Sla  or   SLA 
        Smtp  or  SMTP 
        Sql  or   SQL 
        Ssh  or   SSH 
        Tcp  or   TCP 
        Tls  or   TLS 
        Ttl  or   TTL 
        Udp  or   UDP 
        Ui  or    UI 
        Uid  or   UID 
        Uuid  or  UUID 
        Uri  or   URI 
        Url  or   URL 
        Utf8  or  UTF8 
        Vm  or    VM 
        Xml  or   XML 
        Xmpp  or  XMPP 
        Xsrf  or  XSRF 
        Xss  or   XSS 
