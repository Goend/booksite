1.copy main.go
```
package main

import (
	"fmt"

	"github.com/heimdalr/dag"
)

type Operation struct {
	// ID is the unique identifier of the operation.
	ID int
	// Name is the name of the operation.
	Name string
	// Status is the current status of the operation.
	Status string
}

type OperationVisitor struct {
	DAG *dag.DAG
	// Value is the value of the operation of name.
	Values []string
}

func main() {
	// Initialize a new graph.
	d := dag.NewDAG()

	// Init vertices.
	v0, _ := d.AddVertex(Operation{
		ID:     0,
		Name:   "check net",
		Status: "Success",
	})
	v1, _ := d.AddVertex(Operation{
		ID:     1,
		Name:   "make vpc cni",
		Status: "Success",
	})
	v2, _ := d.AddVertex(Operation{
		ID:     2,
		Name:   "other work",
		Status: "Success",
	})
	v3, _ := d.AddVertex(Operation{
		ID:     3,
		Name:   "make cluster master",
		Status: "Success",
	})
	v4, _ := d.AddVertex(Operation{
		ID:     4,
		Name:   "make cluster work 0-5",
		Status: "Pending",
	})
	v5, _ := d.AddVertex(Operation{
		ID:     5,
		Name:   "make cluster work 6-10",
		Status: "Pending",
	})
	v6, _ := d.AddVertex(Operation{
		ID:     6,
		Name:   "xx6",
		Status: "Failed",
	})

	v7, _ := d.AddVertex(Operation{
		ID:     7,
		Name:   "xx7",
		Status: "Pending",
	})

	// 0
	// | \
	// 1  2
	// | /  \
	// 3     6
	// | \    \
	// 4  5    7
	// Add the above vertices and connect them.
	_ = d.AddEdge(v0, v1)
	_ = d.AddEdge(v0, v2)
	_ = d.AddEdge(v1, v3)
	_ = d.AddEdge(v2, v3)
	_ = d.AddEdge(v3, v4)
	_ = d.AddEdge(v3, v5)
	_ = d.AddEdge(v2, v6)
	_ = d.AddEdge(v6, v7)

	visit := &OperationVisitor{
		DAG: d,
	}

	// find every path's failed and running status vertex
	d.OrderedWalk(visit)
	for _, value := range visit.Values {
		fmt.Println(d.GetVertex(value))
	}
}

func (o *OperationVisitor) Visit(v dag.Vertexer) {
	id, value := v.Vertex()
	// type assertion to get the underlying value
	if value.(Operation).Status != "Success" {
		o.Values = append(o.Values, id)
	}

}

func (o *OperationVisitor) CheckSuccess(v dag.Vertexer) bool {
	id, value := v.Vertex()
	// type assertion to get the underlying value
	if value.(Operation).Status != "Success" {
		return false
	}
	// get all children
	children, err := o.DAG.GetChildren(id)
	if err != nil {
		panic(err)
	}
	// for all children,get the parents of them and not equal to id
	for childrenId, _ := range children {
		parents, err := o.DAG.GetParents(childrenId)
		if err != nil {
			panic(err)
		}
		for parentId, _ := range parents {
			if parentId != id {
				// get the parent's status
				data, err := o.DAG.GetVertex(parentId)
				if err != nil {
					panic(err)
				}
				op := data.(Operation)
				if op.Status != "Success" {
					return false
				}
			}
		}
	}
	return true

}
```

2.修改github.com/heimdalr/dag 包中的/home/jian/go/pkg/mod/github.com/heimdalr/dag@v1.4.0/visitor.go 文件
- 修改Visitor接口定义
```
// Visitor is the interface that wraps the basic Visit method.
// It can use the Visitor and XXXWalk functions together to traverse the entire DAG.
// And access per-vertex information when traversing.
tyVisitor接口定义 interface {
	Visit(Vertexer)
	CheckSuccess(Vertexer) bool
}
```
- 修改OrderedWalk接口
```
// OrderedWalk implements the Topological Sort algorithm to traverse the entire DAG.
// This means that for any edge a -> b, node a will be visited before node b.
func (d *DAG) OrderedWalk(visitor Visitor) {

	d.muDAG.RLock()
	defer d.muDAG.RUnlock()

	queue := llq.New()
	vertices := d.getRoots()
	for _, id := range vertexIDs(vertices) {
		v := vertices[id]
		sv := storableVertex{WrappedID: id, Value: v}
		queue.Enqueue(sv)
	}

	visited := make(map[string]bool, d.getOrder())

Main:
	for !queue.Empty() {
		v, _ := queue.Dequeue()
		sv := v.(storableVertex)

		if visited[sv.WrappedID] {
			continue
		}

		// if the current vertex has any parent that hasn't been visited yet,
		// put it back into the queue, and work on the next element
		parents, _ := d.GetParents(sv.WrappedID)
		for parent := range parents {
			if !visited[parent] {
				queue.Enqueue(sv)
				continue Main
			}
		}
		//入度为0的父节点  检查状态 状态不为success的节点 输出到visitor.Visit 中
		visitor.Visit(sv)

		if !visited[sv.WrappedID] {
			visited[sv.WrappedID] = true
		}

		//question:当父节点的状态为success时，将子节点入队列？ 这里需要检查所有父节点 避免同一个节点有多个父亲节点中靠左的节点成功执行 会将子节点错误的入队列 也就是说 需要确保此父亲节点的所有孩子的父亲节点都已经执行成功
		if visitor.CheckSuccess(sv) {
			vertices, _ := d.getChildren(sv.WrappedID)
			for _, id := range vertexIDs(vertices) {
				v := vertices[id]
				sv := storableVertex{WrappedID: id, Value: v}
				queue.Enqueue(sv)
			}

		}
	}
}
```
3.修改/home/jian/go/pkg/mod/github.com/heimdalr/dag@v1.4.0/marshal.go文件
- 添加接口实现
```
func (mv *marshalVisitor) CheckSuccess(v Vertexer) bool {
	//noting to do
	return true
}
```

直接运行即可