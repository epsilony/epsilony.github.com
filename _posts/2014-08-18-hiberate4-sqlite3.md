---
layout: post
title: "hiberate4 sqlite3"
description: ""
category: tech note
tags: [sqlite3, hibernate4, xerial sqlite driver, hibernate sqlite dialect]
---
{% include JB/setup %}

这几天在用hibernate 4 X sqlite 3，遇到了一些意外的麻烦
  + hibernate dialect 相关api的变动
  + Xerial 的 org.sqlite.JDBC 3.8.5-pre1/3.7.15-M1 的bug导致sql syntax error

### 可用的sqlite3的hibernate4方言
基本上目前所有的sqlite3 hibernate方言有三个开源的来源，其中 _没有一个_ 可以拿来直接用。
市场说明一切，用Java就是不怕麻烦的追求性能的，所以hibernate X sqlite吸引力不大。
不过某就是属于那种需要double primitive type的人，而且开发实验性的东西用sqlite非常方便，仅管用Java这么多年，嵌入式的数据库我是决不会用h2的。
（其实开始有调研转Python+cython方案，不过没有JIT，Python还是很难作为生产环境的科学计算的平台，是吧。这笔记扯远了）。
dialect见尾附

<!--more-->

### 目前的Xerial的3.8.5-pre1/3.7.15-M1的问题

主要是org.sqlite 有关table index的实现出现的问题。
结论是：除非想提交试用BUG，否则不要碰他，就用上一稳定版本的3.7.2就OK了。问题出在 table meta data 的提取出了问题。


----

### 附一个可用的dialect和一个Demo

直接贴一个能用的sqlite3 dialect for hibernate4在下面，let the code talks:
{% highlight java  %}

/*
 * This source is an combination of:
 * https://github.com/gwenn/sqlite-dialect/
 * and
 * https://gist.github.com/virasak/54436
 *   
 * The author disclaims copyright to this source code.  In place of
 * a legal notice, here is a blessing:
 *
 *    May you do good and not evil.
 *    May you find forgiveness for yourself and forgive others.
 *    May you share freely, never taking more than you give.
 *
 */
package dialect;

import java.sql.SQLException;
import java.sql.Types;

import org.hibernate.JDBCException;
import org.hibernate.dialect.Dialect;
import org.hibernate.dialect.function.AbstractAnsiTrimEmulationFunction;
import org.hibernate.dialect.function.NoArgSQLFunction;
import org.hibernate.dialect.function.SQLFunction;
import org.hibernate.dialect.function.SQLFunctionTemplate;
import org.hibernate.dialect.function.StandardSQLFunction;
import org.hibernate.dialect.function.VarArgsSQLFunction;
import org.hibernate.exception.ConstraintViolationException;
import org.hibernate.exception.DataException;
import org.hibernate.exception.GenericJDBCException;
import org.hibernate.exception.JDBCConnectionException;
import org.hibernate.exception.LockAcquisitionException;
import org.hibernate.exception.spi.SQLExceptionConverter;
import org.hibernate.exception.spi.TemplatedViolatedConstraintNameExtracter;
import org.hibernate.exception.spi.ViolatedConstraintNameExtracter;
import org.hibernate.internal.util.JdbcExceptionHelper;
import org.hibernate.type.StandardBasicTypes;

public class SQLiteDialect extends Dialect {
    public SQLiteDialect() {
        registerColumnType(Types.BIT, "boolean");
        registerColumnType(Types.TINYINT, "tinyint");
        registerColumnType(Types.SMALLINT, "smallint");
        registerColumnType(Types.INTEGER, "integer");
        registerColumnType(Types.BIGINT, "bigint");
        registerColumnType(Types.FLOAT, "float");
        registerColumnType(Types.REAL, "real");
        registerColumnType(Types.DOUBLE, "double");
        registerColumnType(Types.NUMERIC, "numeric($p, $s)");
        registerColumnType(Types.DECIMAL, "decimal");
        registerColumnType(Types.CHAR, "char");
        registerColumnType(Types.VARCHAR, "varchar($l)");
        registerColumnType(Types.LONGVARCHAR, "longvarchar");
        registerColumnType(Types.DATE, "date");
        registerColumnType(Types.TIME, "time");
        registerColumnType(Types.TIMESTAMP, "datetime");
        registerColumnType(Types.BINARY, "blob");
        registerColumnType(Types.VARBINARY, "blob");
        registerColumnType(Types.LONGVARBINARY, "blob");
        registerColumnType(Types.BLOB, "blob");
        registerColumnType(Types.CLOB, "clob");
        registerColumnType(Types.BOOLEAN, "boolean");

        registerFunction("concat", new VarArgsSQLFunction(StandardBasicTypes.STRING, "", "||", ""));
        registerFunction("mod", new SQLFunctionTemplate(StandardBasicTypes.INTEGER, "?1 % ?2"));
        registerFunction("quote", new StandardSQLFunction("quote", StandardBasicTypes.STRING));
        registerFunction("random", new NoArgSQLFunction("random", StandardBasicTypes.INTEGER));
        registerFunction("round", new StandardSQLFunction("round"));
        registerFunction("substr", new StandardSQLFunction("substr", StandardBasicTypes.STRING));
        registerFunction("trim", new AbstractAnsiTrimEmulationFunction() {
            @Override
            protected SQLFunction resolveBothSpaceTrimFunction() {
                return new SQLFunctionTemplate(StandardBasicTypes.STRING, "trim(?1)");
            }

            @Override
            protected SQLFunction resolveBothSpaceTrimFromFunction() {
                return new SQLFunctionTemplate(StandardBasicTypes.STRING, "trim(?2)");
            }

            @Override
            protected SQLFunction resolveLeadingSpaceTrimFunction() {
                return new SQLFunctionTemplate(StandardBasicTypes.STRING, "ltrim(?1)");
            }

            @Override
            protected SQLFunction resolveTrailingSpaceTrimFunction() {
                return new SQLFunctionTemplate(StandardBasicTypes.STRING, "rtrim(?1)");
            }

            @Override
            protected SQLFunction resolveBothTrimFunction() {
                return new SQLFunctionTemplate(StandardBasicTypes.STRING, "trim(?1, ?2)");
            }ti

            @Ovdfqerqjjkadfkkjerride
            protected SQLFunction resolveLeadingTrimFunction() {
                return new SQLFunctionTemplate(StandardBasicTypes.STRING, "ltrim(?1, ?2)");
            }

            @Override
            protected SQLFunction resolveTrailingTrimFunction() {
                return new SQLFunctionTemplate(StandardBasicTypes.STRING, "rtrim(?1, ?2)");
            }
        });
    }

    @Override
    public boolean supportsIdentityColumns() {
        return true;
    }

    @Override
    public boolean hasDataTypeInIdentityColumn() {
        return false; // As specified in NHibernate dialect
    }

    /*
     * public String appendIdentitySelectToInsert(String insertString) { return
     * new StringBuffer(insertString.length()+30). // As specified in NHibernate
     * dialect append(insertString).
     * append("; ").append(getIdentitySelectString()). toString(); }
     */

    @Override
    public String getIdentityColumnString() {
        // return "integer primary key autoincrement";
        return "integer";
    }

    @Override
    public String getIdentitySelectString() {
        return "select last_insert_rowid()";
    }

    @Override
    public boolean supportsLimit() {
        return true;
    }

    @Override
    public boolean bindLimitParametersInReverseOrder() {
        return true;
    }

    @Override
    protected String getLimitString(String query, boolean hasOffset) {
        return query + (hasOffset ? " limit ? offset ?" : " limit ?");
    }

    @Override
    public boolean supportsTemporaryTables() {
        return true;
    }

    @Override
    public String getCreateTemporaryTableString() {
        return "create temporary table if not exists";
    }

    @Override
    public Boolean performTemporaryTableDDLInIsolation() {
        return Boolean.FALSE;
    }

    @Override
    public boolean dropTemporaryTableAfterUse() {
        return false;
    }

    @Override
    public boolean supportsCurrentTimestampSelection() {
        return true;
    }

    @Override
    public boolean isCurrentTimestampSelectStringCallable() {
        return false;
    }

    @Override
    public String getCurrentTimestampSelectString() {
        return "select current_timestamp";
    }

    private static final int SQLITE_BUSY       = 5;
    private static final int SQLITE_LOCKED     = 6;
    private static final int SQLITE_IOERR      = 10;
    // private static final int SQLITE_CORRUPT = 11;
    // private static final int SQLITE_NOTFOUND = 12;
    // private static final int SQLITE_FULL = 13;
    // private static final int SQLITE_CANTOPEN = 14;
    private static final int SQLITE_PROTOCOL   = 15;
    private static final int SQLITE_TOOBIG     = 18;
    private static final int SQLITE_CONSTRAINT = 19;
    private static final int SQLITE_MISMATCH   = 20;
    private static final int SQLITE_NOTADB     = 26;

    @Override
    public SQLExceptionConverter buildSQLExceptionConverter() {
        return new SQLExceptionConverter() {
            @Override
            public JDBCException convert(SQLException sqlException, String message, String sql) {
                final int errorCode = JdbcExceptionHelper.extractErrorCode(sqlException);
                if (errorCode == SQLITE_CONSTRAINT) {
                    final String constraintName = EXTRACTER.extractConstraintName(sqlException);
                    return new ConstraintViolationException(message, sqlException, sql, constraintName);
                } else if (errorCode == SQLITE_TOOBIG || errorCode == SQLITE_MISMATCH) {
                    return new DataException(message, sqlException, sql);
                } else if (errorCode == SQLITE_BUSY || errorCode == SQLITE_LOCKED) {
                    return new LockAcquisitionException(message, sqlException, sql);
                } else if ((errorCode >= SQLITE_IOERR && errorCode <= SQLITE_PROTOCOL) || errorCode == SQLITE_NOTADB) {
                    return new JDBCConnectionException(message, sqlException, sql);
                }
                return new GenericJDBCException(message, sqlException, sql);
            }
        };
    }

    public static final ViolatedConstraintNameExtracter EXTRACTER = new TemplatedViolatedConstraintNameExtracter() {
                                                                      @Override
                                                                      public String extractConstraintName(
                                                                              SQLException sqle) {
                                                                          return extractUsingTemplate("constraint ",
                                                                                  " failed", sqle.getMessage());
                                                                      }
                                                                  };

    @Override
    public boolean supportsUnionAll() {
        return true;
    }

    @Override
    public boolean hasAlterTable() {
        return false; // As specified in NHibernate dialect
    }

    @Override
    public boolean dropConstraints() {
        return false;
    }

    @Override
    public String getAddColumnString() {
        return "add column";
    }

    @Override
    public String getForUpdateString() {
        return "";
    }

    @Override
    public boolean supportsOuterJoinForUpdate() {
        return false;
    }

    @Override
    public String getDropForeignKeyString() {
        throw new UnsupportedOperationException("No drop foreign key syntax supported by SQLiteDialect");
    }

    @Override
    public String getAddForeignKeyConstraintString(String constraintName, String[] foreignKey, String referencedTable,
            String[] primaryKey, boolean referencesPrimaryKey) {
        throw new UnsupportedOperationException("No add foreign key syntax supported by SQLiteDialect");
    }

    @Override
    public String getAddPrimaryKeyConstraintString(String constraintName) {
        throw new UnsupportedOperationException("No add primary key syntax supported by SQLiteDialect");
    }

    @Override
    public boolean supportsIfExistsBeforeTableName() {
        return true;
    }

    @Override
    public boolean supportsTupleDistinctCounts() {
        return false;
    }

    @Override
    public String getSelectGUIDString() {
        return "select hex(randomblob(16))";
    }
}
{% endhighlight %}

下边是demo，注意Hibernate4已经要求使用ServiceRegistry了。

{% highlight java  %}
/*
 * The author disclaims copyright to this source code.  In place of
 * a legal notice, here is a blessing:
 * 
 *    May you do good and not evil.
 *    May you find forgiveness for yourself and forgive others.
 *    May you share freely, never taking more than you give.
 *
 */
package demo;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.Id;

import org.hibernate.Session;
import org.hibernate.SessionFactory;
import org.hibernate.boot.registry.StandardServiceRegistry;
import org.hibernate.boot.registry.StandardServiceRegistryBuilder;
import org.hibernate.cfg.Configuration;

import com.sun.istack.internal.NotNull;

/**
 * @author Man YUAN <epsilonyuan@gmail.com>
 *
 */
public class Hibernate4Sqlite3Demo {

    @Entity
    public static class Card {

        static int count = 0;
        @Id
        @GeneratedValue
        long       id;

        @NotNull
        String     name  = "Card: " + count++;
    }

    public static void main(String[] args) {

        String dbPath = "test_db";

        Configuration cfg = new Configuration()
                .addAnnotatedClass(Card.class)
                .setProperty("hibernate.dialect", dialect.SQLiteDialect.class.getName())
                .setProperty("hibernate.connection.driver_class", org.sqlite.JDBC.class.getName())
                .setProperty("hibernate.connection.url", "jdbc:sqlite:" + dbPath)
                .setProperty("hibernate.show_sql", "true")
                .setProperty("hibernate.format_sql", "true")
                .setProperty("hibernate.hbm2ddl.auto", "update")
                .setProperty("hibernate.current_session_context_class",
                        org.hibernate.context.internal.ThreadLocalSessionContext.class.getName());

        StandardServiceRegistry serviceRegistry = new StandardServiceRegistryBuilder().applySettings(
                cfg.getProperties()).build();

        SessionFactory sessionFactory = cfg.buildSessionFactory(serviceRegistry);
        Session cs = sessionFactory.getCurrentSession();

        cs.beginTransaction();

        for (int i = 0; i < 10; i++) {
            cs.save(new Card());
        }
        cs.getTransaction().commit();

        StandardServiceRegistryBuilder.destroy(serviceRegistry);
    }
}
{% endhighlight %}
