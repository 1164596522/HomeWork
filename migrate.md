### 基于lagou_position、s_provinces 分离出 lagou（city、company、position）实现符合规范的职位信息数据：

#### 1.查询 depth 列为 3 得出所有区县后 china_city c 即市进行内链接得出和区县对应的市后再次和 china_city p 即省进行内链接得出和市对应的省，用 union 合并结果集，解决自治区、直辖市、特别行政区这些特殊地域的问题后得出数据根据这些数据创建 lagou_city 表

	create table lagou_city as
	  select d.id, p.cityName as province, c.cityName as city, d.cityName as district
	from (select * from s_provinces where depth = 3) d
	join s_provinces c on d.parentId = c.id and c.depth = 2
	join s_provinces p on c.parentId = p.id and p.depth = 1
	union
	select c.id, p.cityName as province, c.cityName as city, null as district
	from (select * from s_provinces where depth = 2) c
	       join s_provinces p on c.parentId = p.id and p.depth = 1

#### 2.查询 lagou_position 的 company_id、company_short_name、company_full_name、company_size、financestage等字段构成 lagou_company 表

	create table lagou_company as
	  select distinct t.company_id         as cid,
	                  t.company_short_name as short_name,
	                  t.company_full_name  as full_name,
	                  t.company_size       as size,
	                  t.financestage
	  from lagou_position t

#### 3.将公司表字段去除通过内链接进行地区关联创建一个由 city、company 列联动 lagou_city、lagou_company 表的lagou_position表

	create table lagou_position as
	  select pid, c.id as city, company_id as company, position, field, salary_min, salary_max, workyear, education, ptype, pnature, advantage, published_at, updated_at
	  from (select * from lagou where district is null) p
	         join lagou_city c on c.city like concat(p.city, '%') and c.district is null;
	
	insert into lagou_position
	select pid, c.id as city, company_id as company, position, field, salary_min, salary_max, workyear, education, ptype, pnature, advantage, published_at, updated_at
	from (select * from lagou where district is not null) p
	       join lagou_city c on c.city like concat(p.city, '%') and c.district like concat(p.district, '%')

