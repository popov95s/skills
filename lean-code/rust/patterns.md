# Patterns: Rust

Which pattern fits which problem, in Rust. The rules are in [SKILL.md](../SKILL.md) and
[guide.md](guide.md).

| Problem | Pattern | Simpler form that may be enough |
| --- | --- | --- |
| Behaviour differs by kind, and cases call services | [Strategy](#strategy) | Data per kind is a `match` returning values |
| A `bool` parameter picks behaviour | [Named mode](#named-mode) | A `bool` field that states a fact |
| Callers must react to different failures | [Error enum](#error-enum) | `anyhow` at the binary edge |
| Outside strings drive behaviour | [Adapter](#adapter) | One `match` where the string is read once |
| The same integer id is cast everywhere | [Newtype id](#newtype-id) | Keep `usize` when it only indexes a slice |

## Strategy

```rust
pub(crate) trait Quoter {
    fn quote(&self) -> Result<Money, QuoteError>;
}

pub(crate) struct CourierQuoter<'a> {
    pub rates: &'a dyn Fn(&str) -> Result<Rate, QuoteError>,
    pub shipment: &'a CourierShipment,
}

impl Quoter for CourierQuoter<'_> {
    fn quote(&self) -> Result<Money, QuoteError> {
        let rate = (self.rates)(&self.shipment.postcode)?;
        Ok(rate.base + rate.per_kg * self.shipment.weight_kg)
    }
}

pub(crate) fn quoter_for<'a>(shipment: &'a Shipment, apis: &'a Apis) -> Box<dyn Quoter + 'a> {
    match shipment {
        Shipment::Postal(postal) => Box::new(PostalQuoter { table: &apis.postal, shipment: postal }),
        Shipment::Courier(courier) => Box::new(CourierQuoter { rates: &apis.courier_rates, shipment: courier }),
        Shipment::Locker(locker) => Box::new(LockerQuoter { find: &apis.lockers, shipment: locker }),
    }
}
```

One file per quoter, one picker, and the use case takes `&dyn Quoter`. Zero-sized
strategies with nothing to inject return `&'static dyn Trait` (`Language::Rust => &RustProfile`).
With feature-gated kinds, gate the variant and the arm with the same `#[cfg]`.

## Named mode

```rust
pub(crate) enum DiffSource { WorkingTree, Index }

pub(crate) fn changed_paths(root: &Path, base: Option<&str>, source: DiffSource) -> Result<ChangedFiles, GitError> {
    let args: &[&str] = match source {
        DiffSource::WorkingTree => &["diff", "--name-only"],
        DiffSource::Index => &["diff", "--name-only", "--cached"],
    };
    run_git(root, base, args)
}
```

The caller reads `changed_paths(root, None, DiffSource::Index)`, not `changed_paths(root, None, true)`.

## Error enum

```rust
#[derive(Debug)]
pub(crate) enum GitError {
    NotARepo(PathBuf),
    TimedOut,
    Failed { status: i32, stderr: String },
}

impl fmt::Display for GitError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            GitError::NotARepo(path) => write!(f, "{} is not a git repository", path.display()),
            GitError::TimedOut => write!(f, "git did not finish in time"),
            GitError::Failed { status, stderr } => write!(f, "git exited with {status}: {stderr}"),
        }
    }
}

impl std::error::Error for GitError {}
```

The CLI turns it into `anyhow` with `?`. Library callers `match` on it.

## Adapter

```rust
pub(super) fn unit_kind(node_kind: &str) -> Option<UnitKind> {
    match node_kind {
        "function_item" => Some(UnitKind::Function),
        "impl_item" => Some(UnitKind::Impl),
        "trait_item" => Some(UnitKind::Trait),
        _ => None,
    }
}
```

The grammar's strings stop here. Everything after this function matches `UnitKind` with no `_`.

## Newtype id

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub(crate) struct FileId(u32);

impl FileId {
    pub(crate) fn from_index(index: usize) -> Result<Self, TooManyFiles> {
        u32::try_from(index).map(FileId).map_err(|_| TooManyFiles)
    }
}
```

Built once where files are numbered; every other place passes `FileId`.
