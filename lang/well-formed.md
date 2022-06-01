# MiniRust well-formedness requirements

The various syntactic constructs of MiniRust (types, functions, ...) come with well-formedness requirements: certain invariants need to be satisfied for this to be considered a well-formed program.
The idea is that for well-formed programs, the `step` function will never panic.
Those requirements are defined in this file.

Note that `check` functions for testing well-formedness return `Option<()>` rather than `bool` so that we can use `?`.

## Well-formed layouts and types

```rust
impl IntType {
    fn check(self) -> Option<()> {
        if !self.size.bytes().is_power_of_two() { yeet!(); }
    }
}

impl Layout(self) {
    fn check(self) -> Option<()> {
        // Size must be a multiple of alignment.
        if self.size.bytes() % self.align.bytes() != 0 { yeet!(); }
    }
}

impl Type {
    fn check(self) -> Option<()> {
        use Type::*;
        match self {
            Int(int_type) => {
                int_type.check()?;
            }
            Bool | RawPtr { .. } => (),
            Ref { pointee, .. } | Box { pointee } => {
                pointee.check()?;
            }
            Tuple { fields, size, align } => {
                // The fields must not overlap.
                // We check fields in the order of their (absolute) offsets.
                fields.sort_by_key(|(offset, type)| offset);
                let mut last_end = Size::ZERO;
                for (offset, type) in fields {
                    // Recursively check the field type.
                    type.check()?;
                    // And ensure it fits after the one we previously checked.
                    if offset < last_end { yeet!(); }
                    last_end = offset.checked_add(type.size())?;
                }
                // And they must all fit into the size.
                if size < last_end { yeet!(); }
            }
            Array { elem, count } => {
                elem.check()?;
                elem.size().checked_mul(count)?;
            }
            Union { fields, size } => {
                // These may overlap, but they must all fit the size.
                for (offset, type) in fields {
                    type.check()?;
                    if size < offset.checked_add(type.size())? { yeet!(); }
                }
            }
            Enum { variants, size, .. } => {
                for variant in variants {
                    variant.check()?;
                    if size < variant.size() { yeet!(); }
                }
            }
        }
    }
}

impl PlaceType {
    fn check(self) -> Option<()> {
        self.type.check()?;
        self.layout().check()?;
    }
}
```

## Well-formed expressions

```rust
impl Value {
    fn check(self, type: Type) -> Option<()> {
        // For now, we only support integer and boolean literals.
        match (self, type) {
            (Value::Int(i), Type::Int(int_type)) => {
                if !i.in_bounds(int_type.signed, int_type.size) { yeet!(); }
            }
            (Value::Bool(_), Type::Bool) => (),
            _ => yeet!(),
        }
    }
}

impl ValueExpr {
    fn check(self, locals: Map<Local, PlaceType>) -> Option<Type> {
        match self {
            Constant(value, type) => {
                value.check(type)?;
                type
            }
            Load { source, destructive: _ } => {
                let ptype = source.check(locals)?;
                ptype.type
            }
            Ref { target, align, mutbl } => {
                let ptype = target.check(locals)?;
                // If `align > ptype.align`, then this operation is "unsafe"
                // since the reference promises more alignment than what the place
                // guarantees. That is exactly what happens for references
                // to packed fields.
                let pointee = Layout { align, ..ptype.layout() };
                Ref { mutbl, pointee }
            }
            AddrOf { target, mutbl } => {
                let ptype = target.check(locals)?;
                RawPtr { mutbl }
            }
            UnOp { operator, operand } => {
                let operand = operand.check(locals)?;
                match operator {
                    Int(int_op, int_ty) => {
                        if !matches!(operand, Int(_)) { yeet!(); }
                        Int(int_ty)
                    }
                }
            }
            BinOp { operator, left, right } => {
                let left = left.check(locals)?;
                let right = right.check(locals)?;
                match operator {
                    Int(int_op, int_ty) => {
                        if !matches!(left, Int(_)) { yeet!(); }
                        if !matches!(right, Int(_)) { yeet!(); }
                        Int(int_ty)
                    }
                    PtrOffset { inbounds: _ } => {
                        if !matches!(left, Ref { .. } | RawPtr { .. }) { yeet!(); }
                        if !matches!(right, Int(_)) { yeet!(); }
                        left
                    }
                }
            }
        }
    }
}

impl PlaceExpr {
    fn check(self, locals: Map<Local, PlaceType>) -> Option<PlaceType> {
        match self {
            Local(name) => locals.get(name),
            Deref { operand, align } => {
                let type = operand.check(locals)?;
                PlaceType { type, align }
            }
            Field { root, field } => {
                let root = root.check(locals)?;
                let (offset, field_ty) = match root.type {
                    Tuple { fields, .. } => fields.get(field)?,
                    Union { fields, .. } => fields.get(field)?,
                    _ => yeet!(),
                };
                // TODO: I am not sure that that this is a valid PlaceType
                // (specifically, that size is a multiple of align).
                PlaceType {
                    align: root.align.restrict_for_offset(offset),
                    type: field_ty,
                }
            }
            Index { root, index } => {
                let root = root.check(locals)?;
                let index = index.check(locals)?;
                if !matches!(index, Int(_)) { yeet!(); }
                let field_ty = match root.type {
                    Array { elem, .. } => elem,
                    _ => yeet!(),
                };
                // We might be adding a multiple of `field_ty.size`, so we have to
                // lower the alignment compared to `root`. `restrict_for_offset`
                // is good for any multiple of that offset as well.
                // TODO: I am not sure that that this is a valid PlaceType
                // (specifically, that size is a multiple of align).
                PlaceType {
                    align: root.align.restrict_for_offset(field_ty.size()),
                    type: field_ty,
                }
            }
        }
    }
}
```

## Well-formed functions and programs

When checking functions, we track for each program point the set of live locals (and their type) at that point.
To handle cyclic CFGs, we track the set of live locals at the beginning of each basic block.
When we first encounter a block, we add the locals that are live on the "in" edge; when we encounter a block the second time, we require the set to be the same.

```rust
impl Statement {
    /// This returns the adjusted live local mapping after the statement.
    fn check(
        self,
        mut live_locals: Map<Local, PlaceType>,
        func: Function
    ) -> Option<Map<Local, PlaceType>> {
        match self {
            Assign { destination, source } => {
                let left = destination.check(live_locals)?;
                let right = source.check(live_locals)?;
                if left.type != right { yeet!(); }
                locals
            }
            Finalize { place } => {
                place.check(live_locals)?;
                locals
            }
            StorageLive(local) => {
                // Look up the type in the function, and add it to the live locals.
                // Fail if it already is live.
                locals.try_insert(local, func.locals.get(local)?)?;
                locals
            }
            StorageDead(local) => {
                locals.remove(local)?;
                locals
            }
        }
    }
}

impl Terminator {
    /// Returns the successor basic blocks that need to be checked next.
    fn check(
        self,
        live_locals: Map<Local, PlaceType>,
    ) -> Option<List<BbName>> {
        match self {
            Goto(block_name) => {
                list![block_name]
            }
            If { condition, then_block, else_block } => {
                let type = condition.check(live_locals)?;
                if !matches!(type, Type::Bool) { yeet!(); }
                list![then_block, else_block]
            }
            Call { callee: _, arguments, ret, next_block } => {
                // Argument and return expressions must all typecheck with some type.
                for (arg, _abi) in arguments {
                    arg.check(live_locals)?;
                }
                let (ret_place, _ret_abi) = ret;
                ret_place.check(live_locals)?;
                list![next_block]
            }
            Return => {
                list![]
            }
        }
    }
}

impl Function {
    fn check(self) -> Option<()> {
        // Construct initially live locals.
        // Also ensures that argument and return locals must exist.
        let mut start_live: Map<Local, PlaceType> = default();
        for arg in self.args {
            // Also ensures that no two arguments refer to the same local.
            start_live.try_insert(arg, self.locals.get(arg)?)?;
        }
        start_live.try_insert(self.ret, self.locals.get(self.ret)?)?;

        // Check the basic blocks. They can be cyclic, so we keep a worklist of
        // which blocks we still have to check. We also track the live locals
        // they start out with.
        let mut bb_live_at_entry: Map<BbName, Map<Local, PlaceType>> = default();
        bb_live_at_entry.insert(self.start, start_live);
        let mut todo = list![self.start];
        while let Some(block_name) = todo.pop_front() {
            let block = self.blocks.get(block_name)?;
            let mut live_locals = bb_live_at_entry[block_name];
            // Check this block.
            for statement in block.statements {
                live_locals = statement.check(live_locals, self)?;
            }
            let successors = block.terminator.check(live_locals)?;
            for block_name in successors {
                if let Some(precondition) = bb_live_at_entry.get(block_name) {
                    // A block we already visited (or already have in the worklist).
                    // Make sure the set of initially live locals is consistent!
                    if precondition != live_locals { yeet!(); }
                } else {
                    // A new block.
                    bb_live_at_entry.insert(block_name, live_locals);
                    todo.push_back(block_name);
                }
            }
        }

        // Ensure there are no dead blocks that we failed to reach.
        for block_name in self.blocks.keys() {
            if !bb_live_at_entry.contains(block_name) { yeet!(): }
        }
    }
}

impl Program {
    fn check(self) -> Option<()> {
        self.functions.get(start)?;
        for function in self.functions.values() {
            function.check()?;
        }
    }
}
```